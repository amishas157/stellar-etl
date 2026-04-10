# H003: Orderbook snapshots never advance because `UpdateOrderbook` only rebinds a local slice

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each call to `UpdateOrderbook(start, end, ...)` should mutate or replace the caller's orderbook snapshot so later ledger exports reflect the state at `end`. In a multi-ledger batch, snapshots for successive ledger sequences should change when offers are created, modified, or removed.

## Mechanism

`UpdateOrderbook()` receives `orderbook []ingest.Change` by value and ends with `orderbook = changeCache.GetChanges()`. That assignment only updates the function's local slice header; the caller's `startOrderbook` remains unchanged. `StreamOrderbooks()` and `exportOrderbookBatch()` both assume the passed slice advances across ledgers, so they keep copying an old snapshot into every later batch position even when the ledger actually changed.

## Trigger

Run the orderbook pipeline across a range where at least one offer changes after the starting ledger. Compare the exported snapshot for `batchStart` to any later `seq` in the same batch: the later snapshot will still match the original slice instead of reflecting the intervening offer changes.

## Target Code

- `internal/input/orderbooks.go:160-181` — `exportOrderbookBatch()` calls `UpdateOrderbook()` and then copies `startOrderbook` into later ledger slots
- `internal/input/orderbooks.go:194-209` — `UpdateOrderbook()` reassigns `orderbook` locally instead of returning the new slice
- `internal/input/orderbooks.go:212-227` — `StreamOrderbooks()` also assumes `UpdateOrderbook()` advances the shared starting snapshot

## Evidence

There is no return value from `UpdateOrderbook()`, and no pointer or shared container is used to update the caller's slice header. The only state update is `orderbook = changeCache.GetChanges()`, which is the classic Go value-semantics trap for slice parameters.

## Anti-Evidence

If no offers change during the requested range, the bug is invisible because the stale snapshot happens to be correct. Some internal arrays could be reused by the compactor, but the new slice header returned by `GetChanges()` is still discarded by the caller-facing API.
