# 001: export_ledgers captive-core drops Soroban LCM fields

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-input
**Final review by**: gpt-5.4, high

## Summary

`export_ledgers --captive-core` takes the opposite code path from what the flag promises: it reads from the history archive helper instead of the captive-core/datastore ledger backend. That helper never populates `HistoryArchiveLedgerAndLCM.LCM`, so `TransformLedger` silently emits zero values for Soroban-only ledger columns while leaving the rest of the ledger row valid-looking.

## Root Cause

`cmd/export_ledgers.go` special-cases `UseCaptiveCore` and routes it to `input.GetLedgersHistoryArchive()`. `GetLedgersHistoryArchive()` only fills the `Ledger` field of `HistoryArchiveLedgerAndLCM`, leaving `LCM` at its zero value; `TransformLedger()` reads Soroban fields exclusively from `lcm.GetV1()` / `lcm.GetV2()`, so the zero-value close meta causes those fields to be dropped without an error.

## Reproduction

During normal operation, running `stellar-etl export_ledgers --captive-core` over a Soroban-era ledger range produces rows whose classic header fields are correct but whose Soroban-derived fields are zeroed or empty. The same ledger transformed with a populated `LedgerCloseMeta` exports non-zero Soroban values, showing that the corruption comes from the `--captive-core` reader selection rather than from `TransformLedger` rejecting the ledger.

## Affected Code

- `cmd/export_ledgers.go:17-74` — `Run` routes `UseCaptiveCore=true` to `input.GetLedgersHistoryArchive()` and passes the returned zero-value `LCM` into `TransformLedger`
- `internal/input/ledgers_history_archive.go:10-34` — `GetLedgersHistoryArchive` constructs `HistoryArchiveLedgerAndLCM{Ledger: ledger}` without setting `LCM`
- `internal/transform/ledger.go:61-91` — `TransformLedger` populates Soroban-only output fields solely from `lcm.GetV1()` / `lcm.GetV2()`
- `internal/utils/main.go:232-245` — `AddCommonFlags` documents `--captive-core` as using captive core, not the history archive path

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPocZeroLCMDropsSorobanFields`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestPocZeroLCMDropsSorobanFields(t *testing.T) {
	inputs, err := makeLedgerTestInput()
	if err != nil {
		t.Fatalf("makeLedgerTestInput() error = %v", err)
	}

	fullInput := inputs[0]
	fullInput.LCM.V1.TotalByteSizeOfLiveSorobanState = 9876

	fullOutput, err := TransformLedger(fullInput.Ledger, fullInput.LCM)
	if err != nil {
		t.Fatalf("TransformLedger(full LCM) error = %v", err)
	}

	zeroOutput, err := TransformLedger(fullInput.Ledger, xdr.LedgerCloseMeta{})
	if err != nil {
		t.Fatalf("TransformLedger(zero LCM) error = %v", err)
	}

	if fullOutput.Sequence != zeroOutput.Sequence {
		t.Fatalf("expected non-Soroban ledger fields to remain stable, got sequence %d vs %d", fullOutput.Sequence, zeroOutput.Sequence)
	}
	if fullOutput.LedgerHash != zeroOutput.LedgerHash {
		t.Fatalf("expected ledger hash to remain stable, got %q vs %q", fullOutput.LedgerHash, zeroOutput.LedgerHash)
	}

	if fullOutput.SorobanFeeWrite1Kb != 1234 {
		t.Fatalf("expected control SorobanFeeWrite1Kb=1234, got %d", fullOutput.SorobanFeeWrite1Kb)
	}
	if fullOutput.TotalByteSizeOfLiveSorobanState != 9876 {
		t.Fatalf("expected control TotalByteSizeOfLiveSorobanState=9876, got %d", fullOutput.TotalByteSizeOfLiveSorobanState)
	}
	if len(fullOutput.EvictedLedgerKeysType) != 1 || len(fullOutput.EvictedLedgerKeysHash) != 1 {
		t.Fatalf("expected control evicted keys to be populated, got types=%v hashes=%v", fullOutput.EvictedLedgerKeysType, fullOutput.EvictedLedgerKeysHash)
	}

	if zeroOutput.SorobanFeeWrite1Kb != 0 {
		t.Fatalf("expected zero LCM SorobanFeeWrite1Kb=0, got %d", zeroOutput.SorobanFeeWrite1Kb)
	}
	if zeroOutput.TotalByteSizeOfLiveSorobanState != 0 {
		t.Fatalf("expected zero LCM TotalByteSizeOfLiveSorobanState=0, got %d", zeroOutput.TotalByteSizeOfLiveSorobanState)
	}
	if len(zeroOutput.EvictedLedgerKeysType) != 0 || len(zeroOutput.EvictedLedgerKeysHash) != 0 {
		t.Fatalf("expected zero LCM evicted keys to be empty, got types=%v hashes=%v", zeroOutput.EvictedLedgerKeysType, zeroOutput.EvictedLedgerKeysHash)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_ledgers --captive-core` should use captive core and preserve Soroban-derived ledger fields when the underlying close meta contains them.
- **Actual**: the command switches to a history-archive reader that supplies a zero-value `LedgerCloseMeta`, so Soroban-derived output columns become `0` or `[]` even though the ledger row otherwise looks correct.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC holds the ledger constant and changes only the supplied `LedgerCloseMeta`, which is the exact data dependency used by `TransformLedger`.
2. Realistic preconditions: YES — `GetLedgersHistoryArchive()` really returns `HistoryArchiveLedgerAndLCM` values with `LCM` omitted, and `export_ledgers` selects that path when `--captive-core` is set.
3. Bug vs by-design: BUG — the documented meaning of `--captive-core` is to use captive core, while this command instead bypasses `CreateLedgerBackend(..., true, ...)` and loses close-meta fields.
4. Final severity: High — the corruption affects structural ledger-export fields rather than direct monetary amounts, but it silently produces wrong analytics data.
5. In scope: YES — this is a concrete silent data-corruption path in production export code.
6. Test correctness: CORRECT — the assertions are not tautological; they demonstrate that normal ledger fields remain stable while Soroban-only fields disappear.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Remove the special `UseCaptiveCore` branch in `export_ledgers` and always call `input.GetLedgers(..., commonArgs.UseCaptiveCore)`, so the existing ledger-backend factory decides between captive core and datastore without dropping `LedgerCloseMeta`. If a history-archive-only ledger export mode is still needed, it should use a distinct flag and must not expose Soroban-only columns as if they were complete.
