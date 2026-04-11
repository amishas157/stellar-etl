# H002: End-time lookup rounds up and includes a ledger that closed after the requested window

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `get_ledger_range_from_times` receives an `--end-time` that falls between the close times of ledger `N` and ledger `N+1`, the returned `end` ledger should be `N`: the last ledger whose `closed_at` is not later than the requested end instant. Downstream exports seeded from that range should not include ledger `N+1`, because that ledger closed outside the requested time window.

## Mechanism

`GetLedgerRange()` calls the same helper, `findLedgerForDate()`, for both the `start` and `end` boundaries. That helper is explicitly documented as returning the ledger that was closed "on or directly after targetTime," and its guard condition returns the first ledger whose close time is greater than or equal to the target. That ceiling behavior is appropriate for the start boundary but wrong for the end boundary: a whole-second end time that lands between two ledger closes is rounded up to the later ledger, causing the exported range to overrun the requested window by one full ledger.

## Trigger

1. Find consecutive ledgers `N` and `N+1` with close times `T` and `T+Δ`.
2. Run `stellar-etl get_ledger_range_from_times --start-time <time at or before T> --end-time <whole-second instant strictly between T and T+Δ>`.
3. The correct JSON should end at ledger `N`, because ledger `N+1` closed later than the requested end time.
4. The current code path can return `"end": N+1`, and any export run with that range will include rows from a post-window ledger.

## Target Code

- `internal/input/ledger_range.go:GetLedgerRange:31-67` — uses `findLedgerForDate()` for both start and end boundaries
- `internal/input/ledger_range.go:142-159` — helper comment and boundary predicate return the first ledger closed on or after the target time
- `internal/input/ledger_range.go:163-193` — recursive stepping logic preserves that ceiling semantics until a match is found
- `cmd/get_ledger_range_from_times.go:20-25` — exposes the returned range directly as the command's user-facing output

## Evidence

The helper's own comment says it searches for the ledger "closed on or directly after targetTime," and the predicate `prev < target && current >= target` implements exactly that ceiling rule. `GetLedgerRange()` applies that rule symmetrically to the end boundary without any separate "last ledger before end" adjustment. That means a user asking for a time-bounded export can receive a plausible but too-large ledger range even when all timestamps are whole-second values.

## Anti-Evidence

If the command were intentionally documented as returning the smallest ledger range that merely *brackets* the requested timestamps, then a rounded-up end ledger could be defended as superset behavior. The current help text and downstream export usage do not make that superset contract explicit, so the present behavior still looks like silent overexport rather than intentional padding.

---

## Review

**Verdict**: NOT_VIABLE — duplicate of 021-end-time-rounds-up-to-the-next-ledger.md
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of ai-summary/fail/data-input/021-end-time-rounds-up-to-the-next-ledger.md
**Failed At**: reviewer

### Trace Summary

This hypothesis is substantively identical to the previously investigated and rejected hypothesis in `ai-summary/fail/data-input/021-end-time-rounds-up-to-the-next-ledger.md`. Both claim that `findLedgerForDate()` ceiling semantics are wrong for the end boundary. The prior review confirmed that the README (line 333) explicitly documents the contract as "the smallest possible ledger range that completely covers the provided time period," making ceiling semantics for both boundaries the correct implementation.

### Code Paths Examined

- `internal/input/ledger_range.go:GetLedgerRange:32-67` — same code path as prior investigation
- `internal/input/ledger_range.go:findLedgerForDate:142-160` — same ceiling predicate as prior investigation

### Why It Failed

Exact duplicate of a previously investigated hypothesis. The prior review (021) conclusively determined this is working-as-designed behavior per the README's "completely covers" contract.

### Lesson Learned

Check the fail directory for prior investigations before re-submitting the same hypothesis with identical mechanism and target code.
