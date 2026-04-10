# H002: `UpdateOrderbook` loses state updates because it reassigns a slice copy

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Stale orderbook snapshots
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each call to `UpdateOrderbook()` should mutate the caller-visible orderbook so later snapshots reflect offer changes from prior ledgers. `exportOrderbookBatch()` should therefore emit a distinct state per sequence as it walks the range.

## Mechanism

`UpdateOrderbook()` takes `orderbook []ingest.Change` by value and ends with `orderbook = changeCache.GetChanges()`, which only updates its local slice header. `exportOrderbookBatch()` and `StreamOrderbooks()` keep copying the original `startOrderbook`, so every subsequent snapshot can stay anchored to stale state even after new offer changes were compacted.

## Trigger

Call `StreamOrderbooks()` or `exportOrderbookBatch()` over more than one ledger so the orderbook is supposed to evolve between snapshots.

## Target Code

- `internal/input/orderbooks.go:160-191` — `exportOrderbookBatch()` repeatedly copies `startOrderbook` after calling `UpdateOrderbook()`
- `internal/input/orderbooks.go:194-209` — `UpdateOrderbook()` reassigns the local slice instead of updating the caller's state
- `internal/input/orderbooks.go:212-215` — `StreamOrderbooks()` relies on the same by-value update during initial checkpoint alignment

## Evidence

The code has the classic Go value-semantics pattern from the investigation guide: a slice parameter is passed by value and the function rebinds it locally. No pointer return, no `copy()` back into the caller slice, and no returned replacement slice exists anywhere on this path.

## Anti-Evidence

Repository searches did not find any current export command or other in-repo caller that wires `StreamOrderbooks()` into a user-facing dataset, so this stale-state bug does not currently reach the active CLI export surface.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not connect the bug to a reachable export in this repository today. The code path looks broken, but it appears dormant rather than part of a live dataset-producing command.

### Lesson Learned

Orderbook code is still worth tracking because it matches a high-yield Go slice bug pattern. If an orderbook export command is added or restored, this path should be re-opened immediately.
