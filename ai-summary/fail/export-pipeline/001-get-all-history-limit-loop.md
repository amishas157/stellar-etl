# H001: Positive limits leave `GetAllHistory` empty

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`GetAllHistory()` should mirror the sibling bounded readers in `transactions.go`, `operations.go`, and `trades.go`: a positive `limit` should allow reads until the requested number of records is collected or EOF is reached. Calling it with `limit=100` should return up to 100 transactions plus their derived operations and trades.

## Mechanism

The inner transaction loop is written as `for limit < 0`, so it only executes in unbounded mode. Any non-negative limit causes the function to skip every transaction in every ledger and return empty slices even though matching ledger data exists.

## Trigger

Call `input.GetAllHistory(start, end, 100, env, useCaptiveCore)` with any ledger range containing transactions.

## Target Code

- `internal/input/all_history.go:GetAllHistory:21-99` — bounded reader loop uses `for limit < 0`
- `internal/input/transactions.go:GetTransactions:50-67` — sibling uses the expected `len(...) < limit || limit < 0` pattern
- `internal/input/operations.go:GetOperations:52-77` — sibling uses the expected bounded-loop pattern

## Evidence

`GetAllHistory()` is the outlier among the sibling range readers in the same package. The other helpers all continue reading until the slice length reaches the positive limit, while this function never enters the loop at all unless the limit is negative.

## Anti-Evidence

Searches in the repository did not find any current in-repo caller of `GetAllHistory()`, so the broken bounded path does not appear to be wired into an active export command today.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The loop bug is real, but the helper appears unreachable from the current export surface in this repository, so I could not map it to a concrete silently corrupted dataset that a user can request today.

### Lesson Learned

Control-flow bugs in sibling helpers are worth recording, but they should only become live export hypotheses once a reachable command path or production caller is identified.
