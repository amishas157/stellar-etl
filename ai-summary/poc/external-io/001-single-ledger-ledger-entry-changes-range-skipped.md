# H001: Single-ledger `export_ledger_entry_changes` ranges produce no batch at all

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes` is asked to export exactly one ledger (`--start-ledger N --end-ledger N`), it should still emit one batch covering ledger `N` and write the corresponding JSON/Parquet files for the selected entry types.

## Mechanism

`StreamChanges()` computes `batchEnd` as `min(start+batchSize, end)` and then only enters its batching loop while `batchStart < batchEnd`. For a single-ledger range, `batchStart == batchEnd == N`, so the loop never runs, `changeChannel` closes immediately, and `cmd/export_ledger_entry_changes.go` exits without exporting any rows for that ledger.

## Trigger

Run `export_ledger_entry_changes --start-ledger N --end-ledger N --export-accounts true` (or any other export flag) for a ledger known to contain matching entry changes. The command should emit one batch file for ledger `N`, but this code path should instead produce no output files.

## Target Code

- `internal/input/changes.go:162-177` - single-ledger ranges are skipped by `for batchStart < batchEnd`
- `cmd/export_ledger_entry_changes.go:76-89` - command depends entirely on `StreamChanges()` batches to export any rows
- `internal/input/changes_test.go:154-204` - batch-number tests cover multi-ledger and partial ranges, but not `start == end`

## Evidence

`StreamChanges()` initializes `batchStart := start` and `batchEnd := uint32(math.Min(float64(batchStart+batchSize), float64(end)))`. If `start == end`, that yields `batchEnd == batchStart`, so the loop body and its `ExtractBatch()` call are skipped entirely.

## Anti-Evidence

The surrounding batching logic is tested for several non-degenerate ranges and correctly handles partial final batches once the loop starts. This issue only appears at the exact `start == end` boundary, so larger ranges will mask it.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full execution path from `export_ledger_entry_changes` CLI flag parsing through `StreamChanges()` batching logic. When `--start-ledger N --end-ledger N` is specified, `startNum` and `commonArgs.EndNum` are both `N`. These are passed directly to `StreamChanges()` on line 78 of `export_ledger_entry_changes.go`. Inside `StreamChanges()`, `batchEnd` is computed as `min(N + batchSize, N) = N`, making the loop condition `N < N` false. No batches are emitted, the channel closes immediately, and the command exits with zero output. No validation in `ValidateLedgerRange()` or the command itself prevents `start == end`.

### Code Paths Examined

- `internal/input/changes.go:162-178` — `StreamChanges()`: confirmed `batchEnd = uint32(math.Min(float64(start+batchSize), float64(end)))` yields `batchEnd == start` when `start == end`, making the `for batchStart < batchEnd` loop never execute.
- `cmd/export_ledger_entry_changes.go:67-78` — command calls `backend.PrepareRange(ctx, BoundedRange(startNum, commonArgs.EndNum))` (inclusive range, so `BoundedRange(N,N)` is valid requesting one ledger), then passes both values to `StreamChanges()`.
- `cmd/export_ledger_entry_changes.go:72-74` — `EndNum == 0` is special-cased to `MaxInt32` (unbounded), so this bug only manifests when an explicit `--end-ledger` is provided equal to `--start-ledger`.
- `internal/utils/main.go:736-758` — `ValidateLedgerRange()` checks `end < start` but does NOT reject `start == end`, confirming no guard prevents this scenario.
- `internal/input/changes_test.go:154-204` — `TestStreamChangesBatchNumbers` covers ranges of 32, 65, 66, and 128 ledgers but has no test case for `start == end`.
- `internal/input/changes.go:83-158` — `extractBatch()` uses `for seq := batchStart; seq <= batchEnd` which WOULD correctly process a single ledger if called with `batchStart == batchEnd`, but it is never reached.

### Findings

The bug is confirmed. `StreamChanges()` has an off-by-one in its loop condition: it uses strict `<` to compare `batchStart` and `batchEnd`, but `batchEnd` is capped at `end` (inclusive). When the requested range is a single ledger (`start == end`), `batchEnd` equals `batchStart` and the loop body is skipped entirely. The command silently produces no output — no error, no warning — for a valid user request.

The underlying `extractBatch()` function uses `<=` in its inner loop and would correctly handle a single-ledger batch if it were ever called. The fix would be to change the loop condition in `StreamChanges` from `batchStart < batchEnd` to `batchStart <= batchEnd`, with corresponding adjustments to the `batchEnd` decrement logic on line 167 (the `if batchEnd < end` guard) to maintain correct non-overlapping batches for multi-ledger ranges.

### PoC Guidance

- **Test file**: `internal/input/changes_test.go`
- **Setup**: Add a new test case to `TestStreamChangesBatchNumbers` with `batchStart: 5, batchEnd: 5` (single ledger). Use the existing `mockExtractBatch` stub and `batchSize = 64`.
- **Steps**: Call `StreamChanges(nil, 5, 5, 64, changeChan, closeChan, env, logger)` and collect batches from `changeChan`.
- **Assertion**: Assert that exactly one batch is received with `BatchStart: 5, BatchEnd: 5`. Currently, zero batches will be received, demonstrating the bug.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/input/data_integrity_poc_test.go
**Test Name**: "TestSingleLedgerStreamChangesProducesNoBatch"
**Test Language**: Go

### Demonstration

The test calls `StreamChanges(nil, 5, 5, 64, changeChan, closeChan, env, logger)` with a single-ledger range (start=5, end=5) and collects batches from the channel. Zero batches are received, confirming that `StreamChanges` silently skips the entire range. The bug is in the loop condition `batchStart < batchEnd` which evaluates to `5 < 5 = false`, preventing any batch from being emitted for a valid single-ledger request.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestSingleLedgerStreamChangesProducesNoBatch demonstrates that StreamChanges
// silently produces zero batches when start == end (single-ledger range).
func TestSingleLedgerStreamChangesProducesNoBatch(t *testing.T) {
	batchSize := uint32(64)
	changeChan := make(chan ChangeBatch, 10)
	closeChan := make(chan int)
	env := utils.EnvironmentDetails{
		NetworkPassphrase: "",
		ArchiveURLs:       nil,
		BinaryPath:        "",
		CoreConfig:        "",
	}
	logger := utils.NewEtlLogger()
	ExtractBatch = mockExtractBatch

	// Request a single-ledger range: start == end == 5
	go StreamChanges(nil, 5, 5, batchSize, changeChan, closeChan, env, logger)

	var batches []ChangeBatch
	for b := range changeChan {
		batches = append(batches, b)
	}

	// A single-ledger range should produce exactly one batch covering ledger 5.
	// The bug: StreamChanges computes batchEnd = min(5+64, 5) = 5, then the
	// loop condition "batchStart < batchEnd" (5 < 5) is false, so zero batches
	// are emitted.
	if len(batches) == 0 {
		t.Errorf("BUG CONFIRMED: StreamChanges produced 0 batches for single-ledger range (start=5, end=5); expected 1 batch")
	} else if len(batches) != 1 || batches[0].BatchStart != 5 || batches[0].BatchEnd != 5 {
		t.Errorf("Unexpected batch result: got %d batches, first batch start=%d end=%d; expected 1 batch with start=5 end=5",
			len(batches), batches[0].BatchStart, batches[0].BatchEnd)
	} else {
		t.Logf("Single-ledger range correctly produced 1 batch: start=%d end=%d", batches[0].BatchStart, batches[0].BatchEnd)
	}
}
```

### Test Output

```
=== RUN   TestSingleLedgerStreamChangesProducesNoBatch
    data_integrity_poc_test.go:37: BUG CONFIRMED: StreamChanges produced 0 batches for single-ledger range (start=5, end=5); expected 1 batch
--- FAIL: TestSingleLedgerStreamChangesProducesNoBatch (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/input	0.668s
```
