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
