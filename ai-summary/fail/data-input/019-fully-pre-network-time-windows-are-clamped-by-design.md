# H019: Fully pre-network time windows should return empty instead of ledger 2

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a user asks `get_ledger_range_from_times` for a time window that is entirely before Stellar network launch, the command could reasonably be expected to return an empty range or an explicit error, because no ledger in the archive closed during that interval.

## Mechanism

`limitLedgerRange()` clamps both `start` and `end` up to `BeginPoint.CloseTime` whenever they are earlier than the network's lower bound. That means an entirely pre-network interval is normalized into a non-empty in-network request, and the command returns ledger `2` even though the requested window contains no real Stellar history.

## Trigger

Run `stellar-etl get_ledger_range_from_times --start-time <well before network launch> --end-time <also before network launch>`. An "empty" interpretation would produce no usable range, while the current code returns a range anchored at ledger `2`.

## Target Code

- `internal/input/ledger_range.go:GetLedgerRange:31-67` — always searches from ledger `2`
- `internal/input/ledger_range.go:196-208` — clamps both out-of-range endpoints up to `BeginPoint.CloseTime`
- `cmd/get_ledger_range_from_times.go:20-25` — presents the clamped range as the command result

## Evidence

There is no branch that distinguishes "partially overlaps network history" from "entirely precedes network history"; both cases are clamped into the first searchable ledger. That creates a plausible-looking non-empty range for a time window that had no network data at all.

## Anti-Evidence

The command help already describes clamping behavior at the upper bound ("if the time range goes into the future, the ledger range will end on the most recent ledger"), and the codebase intentionally treats ledger `2` as the lower time-search boundary. Existing review on nearby pre-network behavior concluded that this lower-bound clamping is deliberate design, not accidental corruption.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This path follows the command's broader edge-clamping design rather than violating it. The implementation intentionally snaps out-of-range timestamps to the searchable network boundaries, so returning ledger `2` for a fully pre-network request is consistent with that contract even if an empty-range API would also have been reasonable.

### Lesson Learned

Differentiate between an arguably surprising boundary policy and a true correctness defect. When both edges are explicitly clamped into the supported archive range, a surprising non-empty result may still be intentional behavior rather than silent corruption.
