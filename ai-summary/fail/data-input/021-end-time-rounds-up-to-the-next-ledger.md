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

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `GetLedgerRange()` → `findLedgerForDate()` → `findLedgerForTimeBinary()` in `internal/input/ledger_range.go`. Confirmed that both start and end boundaries use ceiling semantics (first ledger with close time ≥ target). Then discovered that the README (line 333) explicitly documents this as the intended contract: "The ledger range that is returned will be the smallest possible ledger range that completely covers the provided time period." Under this "completely covers" contract, ceiling semantics for the end boundary is correct and necessary.

### Code Paths Examined

- `internal/input/ledger_range.go:GetLedgerRange:32-67` — calls `findLedgerForDate()` identically for both start and end
- `internal/input/ledger_range.go:findLedgerForDate:142-160` — predicate `prevTime.CloseTime.Unix() < targetTime.Unix() && currentPoint.CloseTime.Unix() >= targetTime.Unix()` implements ceiling semantics
- `internal/input/ledger_range.go:findLedgerForTimeBinary:101-139` — fallback binary search uses the same ceiling predicate at line 117
- `cmd/get_ledger_range_from_times.go:21-25` — command help text says "Converts a time range into a ledger range"
- `README.md:333` — documents "the smallest possible ledger range that completely covers the provided time period"

### Why It Failed

The hypothesis's expected behavior contradicts the explicitly documented contract. The README states the command returns "the smallest possible ledger range that **completely covers** the provided time period." Under this contract, ceiling semantics for the end boundary is correct: if `endTime` falls between ledger N (close time < endTime) and ledger N+1 (close time > endTime), then ledger N+1 must be included because timestamps in (N.closeTime, endTime] would otherwise be uncovered. The hypothesis's proposed floor behavior (returning N) would produce a range that does NOT completely cover the requested period — any events between N.closeTime and endTime would fall outside the returned range. The `findLedgerForDate()` comment ("closed on or directly after targetTime") and the README documentation are consistent — this is working as designed.

### Lesson Learned

Always check the README/help-text documentation for the explicit behavioral contract before assuming a function's semantics are wrong. The ceiling behavior for both boundaries is the correct implementation of "smallest range that completely covers the time period" — floor semantics for the end would leave a gap.
