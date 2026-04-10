# H003: `UpdateOrderbook()` leaves callers with stale state because it reassigns a slice parameter

**Date**: 2026-04-10
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the orderbook streaming path were part of the live export pipeline, each call to `UpdateOrderbook()` should mutate the caller-visible orderbook so subsequent snapshots reflect new offer changes instead of the original checkpoint state.

## Mechanism

`UpdateOrderbook()` computes a new slice with `changeCache.GetChanges()` and assigns it to the local parameter `orderbook`. Because slices are passed by value, the caller's `startOrderbook` never sees that reassignment, so future snapshots would remain stale.

## Trigger

Invoke `StreamOrderbooks()` or `UpdateOrderbook()` through a caller that expects `startOrderbook` to advance across ledgers.

## Target Code

- `internal/input/orderbooks.go:160-226` — batch exporter repeatedly passes `startOrderbook` into `UpdateOrderbook()`
- `internal/input/orderbooks.go:194-209` — slice is reassigned locally instead of being returned or mutated in place

## Evidence

The code exactly matches the Go value-semantics trap: a slice parameter is rebound inside the callee, while callers keep reusing the original slice.

## Anti-Evidence

`rg` finds no command or other package that currently calls the orderbook streaming APIs, so this stale-state bug does not feed any live JSON/Parquet export in the present repository.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The code path is broken in principle, but it is not wired into a current export command. That makes it a latent bug rather than an active external-I/O integrity issue.

### Lesson Learned

Go slice-reassignment bugs are high-yield, but still need a live caller to matter for current export correctness. For this pipeline, prioritize boundaries that actually emit files today.
