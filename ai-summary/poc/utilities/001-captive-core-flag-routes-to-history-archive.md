# H001: Shared `--captive-core` flag selects the wrong backend in archive-style exports

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a user sets the shared `--captive-core` flag, commands should route through the captive-core / `CreateLedgerBackend` path that `internal/utils` advertises. The exported rows should therefore be built from full `LedgerCloseMeta` input, preserving metadata such as real ledger close times, ledger sequence numbers, and Soroban-only ledger fields.

## Mechanism

`utils.AddCommonFlags()` documents `--captive-core` as "run captive core to retrieve data", and most readers honor that by calling `utils.CreateLedgerBackend(..., useCaptiveCore, ...)`. But `export_assets` and `export_ledgers` invert the meaning of the shared flag: when `UseCaptiveCore` is true they bypass `CreateLedgerBackend` and call history-archive helpers instead, which do not provide the same `LedgerCloseMeta` payload. That means the flag intended to request richer ledger metadata instead selects the poorer input path, producing partial or failed exports.

## Trigger

Run either of these commands with `--captive-core` over a range that contains Soroban ledger metadata or asset-issuing payment / manage-offer operations:

1. `stellar-etl export_ledgers --captive-core -s <start> -e <end>`
2. `stellar-etl export_assets --captive-core -s <start> -e <end>`

Compare the behavior to the default path without `--captive-core`.

## Target Code

- `internal/utils/main.go:AddCommonFlags:231-246` - defines `--captive-core` as the shared "run captive core" selector
- `internal/utils/main.go:MustCommonFlags:460-537` - returns the shared `UseCaptiveCore` value unchanged
- `cmd/export_ledgers.go:28-31` - routes `UseCaptiveCore=true` to `input.GetLedgersHistoryArchive(...)`
- `cmd/export_assets.go:30-33` - routes `UseCaptiveCore=true` to `input.GetPaymentOperationsHistoryArchive(...)`
- `internal/input/ledgers.go:16-25` - the normal path uses `utils.CreateLedgerBackend(...)` and receives real `LedgerCloseMeta`
- `internal/input/assets.go:23-33` - the normal path uses `utils.CreateLedgerBackend(...)` and receives real `LedgerCloseMeta`

## Evidence

The shared flag contract is established in `internal/utils`, but these two commands are the only call sites that branch the opposite way. Their `UseCaptiveCore=true` branch does not invoke `CreateCaptiveCoreBackend` or even `CreateLedgerBackend`; instead it selects history-archive readers that either omit `LCM` entirely or return only history-archive structures.

## Anti-Evidence

If users leave `--captive-core` unset, or use commands that consistently call `CreateLedgerBackend`, the shared flag contract is not violated. The corruption is therefore limited to the two archive-style exports that special-case the flag today.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the flag from `AddCommonFlags` (registers `--captive-core`) through `MustCommonFlags` (reads into `UseCaptiveCore`) into the two anomalous commands. In `export_ledgers.go:28-31`, `UseCaptiveCore=true` routes to `GetLedgersHistoryArchive()` which calls `utils.CreateBackend()` (history archive reader, NOT `CreateLedgerBackend`), returning `HistoryArchiveLedgerAndLCM` with a zero-valued `LCM` field. In `export_assets.go:30-33`, the same flag routes to `GetPaymentOperationsHistoryArchive()` which explicitly sets `LedgerCloseMeta: xdr.LedgerCloseMeta{}`. All 8 other export commands pass `UseCaptiveCore` through to `CreateLedgerBackend()` which correctly dispatches to either captive core or DataStore. These two commands invert the flag, routing to a poorer data source that silently zeroes Soroban-era fields.

### Code Paths Examined

- `cmd/export_ledgers.go:28-31` — `if commonArgs.UseCaptiveCore { GetLedgersHistoryArchive(...) } else { GetLedgers(...) }` — INVERTED: captive-core flag routes to history archive
- `cmd/export_assets.go:30-33` — `if commonArgs.UseCaptiveCore { GetPaymentOperationsHistoryArchive(...) } else { GetPaymentOperations(...) }` — INVERTED: same pattern
- `internal/input/ledgers_history_archive.go:11-35` — calls `utils.CreateBackend(start, end, env.ArchiveURLs)` (history archive); returns `HistoryArchiveLedgerAndLCM{Ledger: ledger}` with LCM zero-valued (line 24-26)
- `internal/input/assets_history_archive.go:13-51` — calls `utils.CreateBackend()`; explicitly sets `LedgerCloseMeta: xdr.LedgerCloseMeta{}` with comment "Using historyArchive will not support getting LCM" (line 38)
- `internal/input/ledgers.go:14-91` — calls `utils.CreateLedgerBackend(ctx, useCaptiveCore, env)` which correctly dispatches; returns both `Ledger` AND `LCM` populated (line 79-82)
- `internal/transform/ledger.go:65-91` — `lcm.GetV1()` and `lcm.GetV2()` on zero LCM return `ok=false`, zeroing `SorobanFeeWrite1Kb`, `TotalByteSizeOfLiveSorobanState`, `EvictedKeysHash`, `EvictedKeysType`
- `internal/transform/asset.go:45-50` — `utils.GetCloseTime(lcm)` and `utils.GetLedgerSequence(lcm)` on zero LCM return epoch time (1970-01-01) and sequence 0
- `internal/utils/main.go:968-976` — `GetCloseTime` and `GetLedgerSequence` call `lcm.LedgerHeaderHistoryEntry()` which for V=0 returns zero-valued header
- `cmd/export_token_transfers.go:28` — contrast: calls `GetLedgers(..., commonArgs.UseCaptiveCore)` unconditionally (correct pattern)
- `cmd/export_transactions.go:25` — contrast: calls `GetTransactions(..., commonArgs.UseCaptiveCore)` (correct pattern)

### Findings

**The flag routing is inverted in exactly 2 of 10 export commands.** When `--captive-core` is set:

1. **`export_ledgers`**: Routes to `GetLedgersHistoryArchive()` → `utils.CreateBackend()` (history archive). The returned `HistoryArchiveLedgerAndLCM` has a zero-valued `LCM`. `TransformLedger` receives this zero LCM, causing all Soroban fields to be silently zeroed: `SorobanFeeWrite1Kb=0`, `TotalByteSizeOfLiveSorobanState=0`, `EvictedKeysHash=nil`, `EvictedKeysType=nil`. For Protocol 20+ ledgers, these should contain real values.

2. **`export_assets`**: Routes to `GetPaymentOperationsHistoryArchive()` → `utils.CreateBackend()`. The code explicitly assigns `LedgerCloseMeta: xdr.LedgerCloseMeta{}` (with a comment acknowledging the limitation). `TransformAsset` calls `GetCloseTime(zero LCM)` → returns epoch time (1970-01-01T00:00:00Z), and `GetLedgerSequence(zero LCM)` → returns 0. Every exported asset gets `ClosedAt=epoch` and `LedgerSequence=0`.

**All 8 other export commands** (`export_transactions`, `export_operations`, `export_effects`, `export_trades`, `export_contract_events`, `export_token_transfers`, `export_ledger_transaction`, `export_ledger_entry_changes`) pass `UseCaptiveCore` through to `CreateLedgerBackend()` which correctly dispatches to captive core or DataStore. The inversion is specific to `export_ledgers` and `export_assets`.

**Note**: The `--captive-core` flag is deprecated ("Will be removed in the Protocol 23 update") and the default DataStore path works correctly. However, when the deprecated flag IS used, the corruption is silent — no error, no warning, valid-looking output with wrong values.

### PoC Guidance

- **Test file**: `internal/input/ledgers_history_archive_test.go` (create new, or append to existing test in `cmd/` if more appropriate)
- **Setup**: Construct a mock scenario where `UseCaptiveCore=true` and compare the output of `GetLedgersHistoryArchive` vs `GetLedgers` for the same ledger range. Alternatively, verify the flag routing directly.
- **Steps**:
  1. Verify `export_ledgers.go:28` branches to `GetLedgersHistoryArchive` when `UseCaptiveCore=true`
  2. Verify `GetLedgersHistoryArchive` returns `HistoryArchiveLedgerAndLCM` with zero-valued `LCM`
  3. Call `TransformLedger` with that zero LCM and a Soroban-era ledger header
  4. Assert Soroban fields are zeroed (demonstrating the corruption)
  5. Similarly for `export_assets.go:30`: verify `GetPaymentOperationsHistoryArchive` returns empty `LedgerCloseMeta`
  6. Call `TransformAsset` with the empty LCM and verify `ClosedAt` and `LedgerSequence` are wrong
- **Assertion**: `TransformLedger` output with zero LCM should have `SorobanFeeWrite1Kb=0` even for a ledger that should have non-zero values; `TransformAsset` output should have `ClosedAt=time.Unix(0,0)` and `LedgerSequence=0`

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestZeroLCMZeroesSorobanFieldsInTransformLedger" and "TestZeroLCMCrashesTransformAsset"
**Test Language**: Go

### Demonstration

Two tests demonstrate the inverted `--captive-core` flag behavior. The first test proves that `TransformLedger` with a zero-valued `LedgerCloseMeta` (as produced by the history-archive path) silently zeroes all Soroban fields (`SorobanFeeWrite1Kb`, `TotalByteSizeOfLiveSorobanState`) on a Protocol 20 ledger that should have non-zero values. The second test proves that `TransformAsset` with a zero-valued `LedgerCloseMeta` (V=0, V0=nil) causes a nil-pointer panic in `GetCloseTime` → `LedgerHeaderHistoryEntry` → `MustV0`, which is even worse than the silent corruption predicted — it crashes the entire export.

### Test Body

```go
package transform

import (
	"testing"
	"time"

	"github.com/stellar/go-stellar-sdk/historyarchive"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestZeroLCMZeroesSorobanFieldsInTransformLedger demonstrates that passing a
// zero-valued LedgerCloseMeta (as happens when the --captive-core flag routes
// export_ledgers to the history-archive path) causes all Soroban-era fields to
// be silently zeroed even when the ledger header represents a Protocol 20+ ledger.
func TestZeroLCMZeroesSorobanFieldsInTransformLedger(t *testing.T) {
	// Construct a valid Soroban-era LCM with non-zero Soroban fields
	realLCM := xdr.LedgerCloseMeta{
		V: 1,
		V1: &xdr.LedgerCloseMetaV1{
			Ext: xdr.LedgerCloseMetaExt{
				V: 1,
				V1: &xdr.LedgerCloseMetaExtV1{
					SorobanFeeWrite1Kb: xdr.Int64(5000),
				},
			},
			TotalByteSizeOfLiveSorobanState: 123456789,
		},
	}

	// The zero-valued LCM that the history-archive path produces
	zeroLCM := xdr.LedgerCloseMeta{}

	// A minimal valid ledger (Protocol 20) — same for both calls
	ledger := historyarchive.Ledger{
		Header: xdr.LedgerHeaderHistoryEntry{
			Header: xdr.LedgerHeader{
				LedgerSeq:     100,
				LedgerVersion: 20,
				TotalCoins:    1000000000,
				FeePool:       100,
				ScpValue: xdr.StellarValue{
					CloseTime: xdr.TimePoint(1700000000),
				},
			},
		},
		Transaction: xdr.TransactionHistoryEntry{
			Ext: xdr.TransactionHistoryEntryExt{V: 0},
		},
		TransactionResult: xdr.TransactionHistoryResultEntry{},
	}

	// Transform with the real LCM — should have Soroban fields populated
	outputReal, err := TransformLedger(ledger, realLCM)
	if err != nil {
		t.Fatalf("TransformLedger with real LCM failed: %v", err)
	}

	// Transform with the zero LCM — simulates the inverted flag path
	outputZero, err := TransformLedger(ledger, zeroLCM)
	if err != nil {
		t.Fatalf("TransformLedger with zero LCM failed: %v", err)
	}

	// With the real LCM, Soroban fields should be populated
	if outputReal.SorobanFeeWrite1Kb != 5000 {
		t.Errorf("Real LCM: SorobanFeeWrite1Kb = %d, want 5000", outputReal.SorobanFeeWrite1Kb)
	}
	if outputReal.TotalByteSizeOfLiveSorobanState != 123456789 {
		t.Errorf("Real LCM: TotalByteSizeOfLiveSorobanState = %d, want 123456789", outputReal.TotalByteSizeOfLiveSorobanState)
	}

	// With the zero LCM, Soroban fields are silently zeroed — this is the bug.
	if outputZero.SorobanFeeWrite1Kb != 0 {
		t.Errorf("Zero LCM: SorobanFeeWrite1Kb = %d, want 0 (demonstrating silent zeroing)", outputZero.SorobanFeeWrite1Kb)
	}
	if outputZero.TotalByteSizeOfLiveSorobanState != 0 {
		t.Errorf("Zero LCM: TotalByteSizeOfLiveSorobanState = %d, want 0 (demonstrating silent zeroing)", outputZero.TotalByteSizeOfLiveSorobanState)
	}

	t.Logf("DEMONSTRATED: Protocol %d ledger (seq=%d) with zero LCM silently loses Soroban fields:",
		outputZero.ProtocolVersion, outputZero.Sequence)
	t.Logf("  SorobanFeeWrite1Kb: real=%d, zero-LCM=%d",
		outputReal.SorobanFeeWrite1Kb, outputZero.SorobanFeeWrite1Kb)
	t.Logf("  TotalByteSizeOfLiveSorobanState: real=%d, zero-LCM=%d",
		outputReal.TotalByteSizeOfLiveSorobanState, outputZero.TotalByteSizeOfLiveSorobanState)
}

// TestZeroLCMCrashesTransformAsset demonstrates that passing a zero-valued
// LedgerCloseMeta (as happens when the --captive-core flag routes export_assets
// to the history-archive path) causes a nil-pointer panic in TransformAsset.
func TestZeroLCMCrashesTransformAsset(t *testing.T) {
	destAccountID := xdr.MustAddress("GBVVRXLMNCJQW3IDDXC3X6XCH35B5Q7QXNMMFPENSOGUPQO7WO7HGZPA")
	dest := destAccountID.ToMuxedAccount()
	paymentOp := xdr.Operation{
		Body: xdr.OperationBody{
			Type: xdr.OperationTypePayment,
			PaymentOp: &xdr.PaymentOp{
				Destination: dest,
				Asset: xdr.Asset{
					Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
					AlphaNum4: &xdr.AlphaNum4{
						AssetCode: xdr.AssetCode4{'U', 'S', 'D', 'C'},
						Issuer:    xdr.MustAddress("GBVVRXLMNCJQW3IDDXC3X6XCH35B5Q7QXNMMFPENSOGUPQO7WO7HGZPA"),
					},
				},
				Amount: 10000000,
			},
		},
	}

	ledgerSeq := int32(50000000)

	// First, verify the real LCM path works correctly
	realLCM := xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					LedgerSeq: xdr.Uint32(ledgerSeq),
					ScpValue: xdr.StellarValue{
						CloseTime: xdr.TimePoint(1700000000),
					},
				},
			},
		},
	}

	outputReal, err := TransformAsset(paymentOp, 0, 0, ledgerSeq, realLCM)
	if err != nil {
		t.Fatalf("TransformAsset with real LCM failed: %v", err)
	}

	expectedTime := time.Unix(1700000000, 0).UTC()
	if !outputReal.ClosedAt.Equal(expectedTime) {
		t.Errorf("Real LCM: ClosedAt = %v, want %v", outputReal.ClosedAt, expectedTime)
	}
	if outputReal.LedgerSequence != uint32(ledgerSeq) {
		t.Errorf("Real LCM: LedgerSequence = %d, want %d", outputReal.LedgerSequence, ledgerSeq)
	}

	// The zero LCM that the history-archive path produces (V=0, V0=nil)
	zeroLCM := xdr.LedgerCloseMeta{}

	panicked := false
	func() {
		defer func() {
			if r := recover(); r != nil {
				panicked = true
				t.Logf("DEMONSTRATED: TransformAsset panics with zero LCM: %v", r)
			}
		}()
		TransformAsset(paymentOp, 0, 0, ledgerSeq, zeroLCM)
	}()

	if !panicked {
		t.Error("Expected TransformAsset to panic with zero LCM, but it did not")
	}

	t.Logf("DEMONSTRATED: export_assets --captive-core routes to history-archive path which sets LedgerCloseMeta: xdr.LedgerCloseMeta{}")
	t.Logf("  This zero LCM (V=0, V0=nil) causes a nil-pointer panic in GetCloseTime -> LedgerHeaderHistoryEntry -> MustV0")
	t.Logf("  Real LCM correctly produces: ClosedAt=%v, LedgerSequence=%d", outputReal.ClosedAt, outputReal.LedgerSequence)
}
```

### Test Output

```
=== RUN   TestZeroLCMZeroesSorobanFieldsInTransformLedger
    data_integrity_poc_test.go:82: DEMONSTRATED: Protocol 20 ledger (seq=100) with zero LCM silently loses Soroban fields:
    data_integrity_poc_test.go:84:   SorobanFeeWrite1Kb: real=5000, zero-LCM=0
    data_integrity_poc_test.go:86:   TotalByteSizeOfLiveSorobanState: real=123456789, zero-LCM=0
--- PASS: TestZeroLCMZeroesSorobanFieldsInTransformLedger (0.00s)
=== RUN   TestZeroLCMCrashesTransformAsset
    data_integrity_poc_test.go:157: DEMONSTRATED: TransformAsset panics with zero LCM: runtime error: invalid memory address or nil pointer dereference
    data_integrity_poc_test.go:167: DEMONSTRATED: export_assets --captive-core routes to history-archive path which sets LedgerCloseMeta: xdr.LedgerCloseMeta{}
    data_integrity_poc_test.go:168:   This zero LCM (V=0, V0=nil) causes a nil-pointer panic in GetCloseTime -> LedgerHeaderHistoryEntry -> MustV0
    data_integrity_poc_test.go:169:   Real LCM correctly produces: ClosedAt=2023-11-14 22:13:20 +0000 UTC, LedgerSequence=50000000
--- PASS: TestZeroLCMCrashesTransformAsset (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.814s
```
