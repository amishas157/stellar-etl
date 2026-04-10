# H002: `GetAllHistory()` silently returns empty results for non-negative limits

**Date**: 2026-04-10
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a live export path uses `GetAllHistory(start, end, limit, ...)` with `limit >= 0`, it should behave like sibling readers and return up to `limit` transactions, operations, and trades instead of empty slices.

## Mechanism

`GetAllHistory()` uses `for limit < 0` where sibling readers use `for len(slice) < limit || limit < 0`. That means any non-negative limit would skip the transaction-reading loop entirely and produce plausible-but-empty outputs.

## Trigger

Call `GetAllHistory()` with `limit = 0` or any positive limit.

## Target Code

- `internal/input/all_history.go:23-99` — outlier limit loop
- `internal/input/transactions.go:50-67` — sibling transaction reader behavior
- `internal/input/operations.go:52-77` — sibling operation reader behavior
- `internal/input/trades.go:50-81` — sibling trade reader behavior

## Evidence

The loop condition in `all_history.go` is inconsistent with every nearby reader and would indeed produce empty slices for non-negative limits.

## Anti-Evidence

`rg` only finds the function definition; no current command or export path in this repository calls `GetAllHistory()`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The bug exists in an internal helper, but it is not currently reachable from the live JSON/Parquet export commands that define this objective. Without a caller, it does not yet produce silent data corruption in shipped export output.

### Lesson Learned

Control-flow outliers are worth flagging, but reachability matters. For this objective, a broken helper without a call site is weaker than an active export boundary that already emits wrong files.
