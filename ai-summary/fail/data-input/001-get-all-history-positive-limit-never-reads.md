# H001: GetAllHistory Positive Limits Never Read Any Transactions

**Date**: 2026-04-10
**Subsystem**: data-input
**Severity**: High
**Impact**: Empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`GetAllHistory()` should read transactions for the requested ledger range until it reaches EOF or the caller-provided positive limit, then return populated `Operations`, `Trades`, and `Ledgers` slices.

## Mechanism

The inner reader loop is `for limit < 0`, so any non-negative limit skips transaction reads entirely and returns empty slices. That is the opposite of the sibling reader pattern, where negative means "read everything" and positive values cap the output.

## Trigger

Call `GetAllHistory(start, end, 1, env, useCaptiveCore)` on any ledger range containing transactions. The correct result should contain at least one transaction and its operations, but the current code returns empty slices.

## Target Code

- `internal/input/all_history.go:23-99` — positive-limit path never enters the transaction read loop

## Evidence

`GetTransactions()`, `GetOperations()`, and `GetTrades()` all use `for int64(len(slice)) < limit || limit < 0`, while `GetAllHistory()` uses only `for limit < 0`. A repository-wide search found no in-tree callers beyond the function definition itself.

## Anti-Evidence

The function can still return data when the caller passes a negative limit, so the implementation is not completely inert. The bug is real at the function level, but I could not tie it to a shipped CLI export in this repository.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`GetAllHistory()` is currently unwired: a full-repo search found no command or other production caller that invokes it, so this control-flow bug does not presently corrupt a user-facing export in `stellar-etl`.

### Lesson Learned

Dead or orphaned reader helpers can contain real bugs, but for this objective they need a live export path before they qualify as data-integrity findings. Always confirm that a suspicious helper is actually reachable from a shipped command.
