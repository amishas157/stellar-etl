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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`UpdateOrderbook` at line 195 takes `orderbook []ingest.Change` by value. It compacts the existing orderbook with new ledger changes via `GetOfferChanges` and `changeCache.AddChange`, then reassigns `orderbook = changeCache.GetChanges()` at line 208. This reassignment only updates the function-local slice header — Go passes slice headers (pointer, length, capacity) by value. The caller's `startOrderbook` variable in both `exportOrderbookBatch` (line 177) and `StreamOrderbooks` (line 215) is never modified. Subsequent iterations in `exportOrderbookBatch` copy the unchanged `startOrderbook` into the batch map, producing stale snapshots for every ledger after the first.

### Code Paths Examined

- `internal/input/orderbooks.go:194-209` — `UpdateOrderbook` receives `orderbook` by value and reassigns it locally at line 208; the new compacted slice is discarded when the function returns
- `internal/input/orderbooks.go:160-192` — `exportOrderbookBatch` calls `UpdateOrderbook(prevSeq, curSeq, startOrderbook, ...)` at line 177, then copies `startOrderbook` (unchanged) into `batchMap[curSeq]` at lines 178-179; every ledger in the batch gets the same stale snapshot
- `internal/input/orderbooks.go:212-237` — `StreamOrderbooks` calls `UpdateOrderbook(checkpointSeq, start, startOrderbook, ...)` at line 215, then passes the unchanged `startOrderbook` to `exportOrderbookBatch` for every batch in the range
- `ChangeCompactor.GetChanges()` returns a new `[]Change` slice — confirmed in upstream SDK `ingest/change_compactor.go`

### Findings

The bug has two manifestation points:

1. **Within a batch** (`exportOrderbookBatch`, lines 166-183): The for-loop iterates from `batchStart+1` to `batchEnd`. Each iteration calls `UpdateOrderbook` which computes the correct updated orderbook but discards the result. The subsequent `copy(batchMap[curSeq], startOrderbook)` copies the original stale snapshot. Every ledger sequence in the batch map (except `batchStart`) contains the orderbook state as of `batchStart`, not the actual state at that ledger.

2. **Across batches** (`StreamOrderbooks`, lines 212-237): The initial `UpdateOrderbook` call at line 215 is supposed to advance `startOrderbook` from the checkpoint sequence to the start of the range. Since it doesn't modify the caller's variable, the first batch already starts with a stale snapshot (the checkpoint state, not `start` state). All subsequent batches reuse this same stale `startOrderbook`.

This is a textbook Go value-semantics bug (Investigation Pattern 4). The function would need to either return the new slice or accept a `*[]ingest.Change` pointer parameter.

### PoC Guidance

- **Test file**: `internal/input/orderbooks_test.go` (create if not exists, or `internal/input/data_integrity_poc_test.go`)
- **Setup**: Create a mock `ChangeCompactor` scenario: construct an initial `[]ingest.Change` with one offer, then simulate `UpdateOrderbook` being called. Since `UpdateOrderbook` depends on `CaptiveStellarCore`, the PoC can directly demonstrate the value-semantics issue by:
  1. Creating a `[]ingest.Change` slice with known content
  2. Passing it to a function that mimics `UpdateOrderbook`'s reassignment pattern
  3. Asserting the caller's variable is unchanged after the call
- **Steps**: 
  1. Create `original := []ingest.Change{...}` with one offer entry
  2. Call a wrapper that does `param = newSlice` (mimicking line 208)
  3. Assert `len(original)` and content are unchanged — proving the update was lost
- **Assertion**: `assert.Equal(t, originalLen, len(callerSlice))` — the caller's slice length and content should be unchanged after `UpdateOrderbook`, proving the updated orderbook is silently discarded
- **Alternative approach**: Call `UpdateOrderbook` directly with a mock `CaptiveStellarCore` that returns known changes, then verify `startOrderbook` was NOT updated — this directly proves the production bug
