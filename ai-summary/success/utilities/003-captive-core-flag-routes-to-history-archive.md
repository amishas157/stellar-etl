# 003: Captive-core flag routes to history archive

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: utilities
**Final review by**: gpt-5.4, high

## Summary

The shared `--captive-core` flag is documented as selecting captive-core-backed ledger reads, but `export_ledgers` and `export_assets` branch the opposite way. In `export_ledgers`, that silently drops `LedgerCloseMeta` and zeroes Soroban-era ledger fields; in `export_assets`, it passes an empty `LedgerCloseMeta` into `TransformAsset`, which panics while reading close time and ledger sequence.

## Root Cause

`cmd/export_ledgers.go` and `cmd/export_assets.go` special-case `UseCaptiveCore=true` by calling history-archive readers instead of the normal `CreateLedgerBackend(..., useCaptiveCore, ...)` path used by the rest of the exporters. Those history-archive readers either leave `LCM` unset or explicitly assign `xdr.LedgerCloseMeta{}`, but the downstream transforms still assume real ledger-close metadata is present.

## Reproduction

Use the public `--captive-core` flag on either affected command. `export_ledgers --captive-core` feeds a zero-valued `LCM` into `TransformLedger`, so Protocol 20+ Soroban fields export as zero values. `export_assets --captive-core` feeds `xdr.LedgerCloseMeta{}` into `TransformAsset`, which panics when `utils.GetCloseTime()` calls `LedgerHeaderHistoryEntry()` on the unset union arm.

## Affected Code

- `cmd/export_ledgers.go:17-32` — `UseCaptiveCore=true` selects `input.GetLedgersHistoryArchive(...)`
- `internal/input/ledgers_history_archive.go:10-34` — returns `HistoryArchiveLedgerAndLCM` with an unset `LCM`
- `internal/transform/ledger.go:61-91` — Soroban-only fields are populated only from `lcm.GetV1()` / `lcm.GetV2()`
- `cmd/export_assets.go:17-34` — `UseCaptiveCore=true` selects `input.GetPaymentOperationsHistoryArchive(...)`
- `internal/input/assets_history_archive.go:13-39` — explicitly sets `LedgerCloseMeta: xdr.LedgerCloseMeta{}`
- `internal/transform/asset.go:45-50` — reads close time and sequence from the supplied `LedgerCloseMeta`
- `internal/utils/main.go:968-975` — `GetCloseTime()` / `GetLedgerSequence()` dereference `lcm.LedgerHeaderHistoryEntry()`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestZeroLCMZeroesSorobanFieldsInTransformLedger` / `TestZeroLCMCrashesTransformAsset`
- **Test language**: go
- **How to run**: Append the test body below to the target test file, then run `go build ./... && go test ./internal/transform/... -run 'TestZeroLCMZeroesSorobanFieldsInTransformLedger|TestZeroLCMCrashesTransformAsset' -v`.

### Test Body

```go
package transform

import (
	"testing"
	"time"

	"github.com/stellar/go-stellar-sdk/historyarchive"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestZeroLCMZeroesSorobanFieldsInTransformLedger(t *testing.T) {
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

	zeroLCM := xdr.LedgerCloseMeta{}

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

	outputReal, err := TransformLedger(ledger, realLCM)
	if err != nil {
		t.Fatalf("TransformLedger with real LCM failed: %v", err)
	}

	outputZero, err := TransformLedger(ledger, zeroLCM)
	if err != nil {
		t.Fatalf("TransformLedger with zero LCM failed: %v", err)
	}

	if outputReal.SorobanFeeWrite1Kb != 5000 {
		t.Fatalf("real LCM SorobanFeeWrite1Kb = %d, want 5000", outputReal.SorobanFeeWrite1Kb)
	}
	if outputReal.TotalByteSizeOfLiveSorobanState != 123456789 {
		t.Fatalf("real LCM TotalByteSizeOfLiveSorobanState = %d, want 123456789", outputReal.TotalByteSizeOfLiveSorobanState)
	}

	if outputZero.SorobanFeeWrite1Kb != 0 {
		t.Fatalf("zero LCM SorobanFeeWrite1Kb = %d, want 0", outputZero.SorobanFeeWrite1Kb)
	}
	if outputZero.TotalByteSizeOfLiveSorobanState != 0 {
		t.Fatalf("zero LCM TotalByteSizeOfLiveSorobanState = %d, want 0", outputZero.TotalByteSizeOfLiveSorobanState)
	}
}

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
		t.Fatalf("real LCM ClosedAt = %v, want %v", outputReal.ClosedAt, expectedTime)
	}
	if outputReal.LedgerSequence != uint32(ledgerSeq) {
		t.Fatalf("real LCM LedgerSequence = %d, want %d", outputReal.LedgerSequence, ledgerSeq)
	}

	zeroLCM := xdr.LedgerCloseMeta{}
	panicked := false
	func() {
		defer func() {
			if recover() != nil {
				panicked = true
			}
		}()
		_, _ = TransformAsset(paymentOp, 0, 0, ledgerSeq, zeroLCM)
	}()

	if !panicked {
		t.Fatal("expected TransformAsset to panic with zero LCM")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Setting `--captive-core` should select the normal captive-core-backed ledger reader, preserving `LedgerCloseMeta` for both exporters.
- **Actual**: `export_ledgers --captive-core` silently zeroes Soroban-only fields, and `export_assets --captive-core` crashes because the selected history-archive reader supplies an empty `LedgerCloseMeta`.

## Adversarial Review

1. Exercises claimed bug: YES — the tests reproduce the exact downstream consequences of the empty `LedgerCloseMeta` values these history-archive readers inject, and the command code shows `--captive-core` selects those readers.
2. Realistic preconditions: YES — the only requirement is invoking the documented public `--captive-core` flag on `export_ledgers` or `export_assets`.
3. Bug vs by-design: BUG — the shared flag help text, README, and `CreateLedgerBackend()` all define `--captive-core` as the selector for captive-core-backed reads, not history-archive fallbacks with missing metadata.
4. Final severity: High — `export_ledgers` silently corrupts structural Soroban ledger fields; the `export_assets` panic is an additional operational failure, but the highest validated impact is High.
5. In scope: YES — this is a public exporter path that returns wrong output or crashes without requiring compromised upstream data.
6. Test correctness: CORRECT — the assertions compare behavior with real versus empty `LedgerCloseMeta` values and exercise the production transform functions that the affected commands call.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Remove the inverted special cases in `cmd/export_ledgers.go` and `cmd/export_assets.go` so both commands always use the standard readers (`GetLedgers` / `GetPaymentOperations`) and simply pass `commonArgs.UseCaptiveCore` through to `CreateLedgerBackend()`. If a history-archive-only fallback is still needed, gate it behind a separate flag with explicit user-facing documentation and hard fail before calling transforms that require `LedgerCloseMeta`.
