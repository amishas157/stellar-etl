# H018: Equal close times could create a boundary gap in ledger-range search

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If two consecutive ledgers were allowed to share the same close time second, `get_ledger_range_from_times` should still return a stable boundary without skipping one of those ledgers or failing to terminate its search.

## Mechanism

Both `findLedgerForDate()` and `findLedgerForTimeBinary()` use a strict `prev.CloseTime.Unix() < targetTime.Unix()` guard. At first glance, that looks dangerous because equal close times would make the "previous < target <= current" interval disappear, potentially forcing the recursion past the correct boundary or making the binary search oscillate.

## Trigger

Query `get_ledger_range_from_times` for a timestamp equal to the shared close-time second of two consecutive ledgers. If equal close times were legal, the correct output would still need to pick a deterministic ledger without creating a search gap.

## Target Code

- `internal/input/ledger_range.go:117-122` — binary-search boundary check uses strict `<` on the previous ledger's close time
- `internal/input/ledger_range.go:158-159` — heuristic search uses the same strict `<` predicate

## Evidence

The boundary logic assumes the searchable close-time sequence is strictly increasing. If that assumption were false and two adjacent ledgers shared the same second, there would be no interval satisfying `prev < target <= current` for the later ledger.

## Anti-Evidence

Stellar documentation describes ledger close time as strictly monotonic, so consecutive ledgers should not share the same close time. A web check against Stellar's ledger docs confirmed that SCP close time selection must move forward rather than reuse the previous ledger's timestamp.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The protocol assumption the code relies on is valid: consecutive Stellar ledgers do not legitimately share the same close time. Without equal adjacent close times on real chain data, the strict `<` guard does not create an exploitable boundary gap.

### Lesson Learned

Before treating a comparison predicate as a range bug, verify the ordering guarantees that the upstream protocol gives the data. A pattern that would be unsafe under duplicate timestamps can still be correct when the source system guarantees strict monotonicity.
