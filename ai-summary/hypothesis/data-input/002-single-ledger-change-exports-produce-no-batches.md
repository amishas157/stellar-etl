# H002: Single-Ledger Change Exports Produce No Batches

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledger_entry_changes --start-ledger L --end-ledger L` should emit one `ChangeBatch` covering ledger `L`, so any account, trustline, offer, pool, contract, or config-setting changes in that ledger are exported instead of silently disappearing.

## Mechanism

`StreamChanges()` initializes `batchEnd` with `min(batchStart+batchSize, end)` and then loops only while `batchStart < batchEnd`. When `start == end`, `batchEnd` equals `batchStart`, so the loop body never runs, `changeChannel` is closed immediately, and the caller exits without receiving a single batch. The command therefore succeeds with an empty output set for a valid one-ledger range.

## Trigger

Run `stellar-etl export_ledger_entry_changes --start-ledger <L> --end-ledger <L> ...` for any ledger known to contain entry changes. The correct behavior is one output batch for `<L>`, but the current implementation closes the channel without sending anything.

## Target Code

- `internal/input/changes.go:162-177` — computes `batchStart`/`batchEnd` and skips the loop entirely when `start == end`
- `cmd/export_ledger_entry_changes.go:76-87` — waits on `changeChan` / `closeChan` and exits cleanly when no batch is ever sent
- `internal/input/changes_test.go:142-229` — exercises 32-, 65-, 66-, and 128-ledger ranges but never covers the `start == end` case

## Evidence

For a one-ledger request, `batchStart` and `batchEnd` are both the same ledger number before the first iteration, so the `<` guard is false immediately. The command-side select loop treats the closed channel as normal completion, which means users get an apparently successful run with silently missing rows.

## Anti-Evidence

Multi-ledger ranges do work because the first computed `batchEnd` is greater than `batchStart`, so the bug hides unless callers request exactly one ledger. The existing tests cover several seam cases, which may have masked the missing single-ledger case by making the batching logic look better exercised than it really is.
