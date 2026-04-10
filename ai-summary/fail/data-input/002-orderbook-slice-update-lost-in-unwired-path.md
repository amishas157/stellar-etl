# H002: Orderbook Snapshot Updates Are Lost Because the Slice Is Reassigned Locally

**Date**: 2026-04-10
**Subsystem**: data-input
**Severity**: High
**Impact**: Stale state instead of updated snapshots
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`StreamOrderbooks()` should advance the caller's `startOrderbook` snapshot from the checkpoint ledger to the requested start ledger and then from ledger to ledger within each batch, so each exported orderbook reflects the actual state at that sequence.

## Mechanism

`UpdateOrderbook()` reassigns its `orderbook` parameter with `changeCache.GetChanges()`, but the slice is passed by value, so the caller's `startOrderbook` variable is never updated. `exportOrderbookBatch()` then copies the original slice into every `batchMap` entry, which would leave downstream consumers with stale snapshots.

## Trigger

Invoke `StreamOrderbooks()` with a `startOrderbook` snapshot from a checkpoint and a later `start` or `end` range where offers change between ledgers. The expected result is a sequence of evolving orderbooks, but the current code would keep copying the unchanged input slice.

## Target Code

- `internal/input/orderbooks.go:160-191` — copies `startOrderbook` into every batch entry after calling `UpdateOrderbook()`
- `internal/input/orderbooks.go:194-215` — recomputes `orderbook` only in a local parameter

## Evidence

The function has the classic Go value-semantics bug: it does not return the updated slice or mutate the caller's slice in place, yet the surrounding code behaves as if it does. A full-repo search found no in-tree callers beyond the orderbook helpers themselves.

## Anti-Evidence

If this code path were wired into an exporter, the stale snapshots would be serious. In the current repository state, however, I could not find any command that calls `StreamOrderbooks()` or `ReceiveParsedOrderbooks()`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The orderbook pipeline is currently unwired in this repository: no command invokes `StreamOrderbooks()` or consumes `OrderbookBatch`, so the bug does not presently affect any shipped JSON or Parquet export.

### Lesson Learned

For data-integrity work, an internal logic bug is only actionable when it feeds a real output surface. Confirm reachability before escalating isolated helper defects into pipeline findings.
