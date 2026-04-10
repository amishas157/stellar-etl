# H003: StreamChanges Batch End Adjustment Might Duplicate Boundary Ledgers

**Date**: 2026-04-10
**Subsystem**: data-input
**Severity**: High
**Impact**: Duplicate ledger-entry-change rows across batch files
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`StreamChanges()` should partition the requested ledger range into non-overlapping batches, so each ledger contributes its change rows to exactly one batch file.

## Mechanism

At first glance, `batchEnd := min(batchStart+batchSize, end)` followed by `if batchEnd < end { batchEnd = batchEnd - 1 }` looks like an off-by-one bug that could either skip or duplicate a boundary ledger. If the boundary math were wrong, downstream `export_ledger_entry_changes` runs would emit duplicate or missing rows at batch seams.

## Trigger

Export ledger entry changes with a batch size such as `64` over a range like `1..66` or `1..128`, where batch seams are exercised. Check whether the seam ledger appears twice or not at all.

## Target Code

- `internal/input/changes.go:162-178` — computes batch boundaries and advances `batchStart`
- `internal/input/changes_test.go:142-229` — codifies expected batch ranges for seam cases

## Evidence

The boundary math is unusual because `batchEnd` is sometimes treated as inclusive and sometimes initialized from an exclusive-style expression. Without the tests, it is easy to suspect duplicate or skipped seam ledgers.

## Anti-Evidence

The tests explicitly expect `1..64` then `65..66` for the seam case, and the implementation advances `batchStart` to `batchEnd + 1`, which preserves non-overlap. I did not find a concrete range where a ledger is emitted twice or omitted.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The inclusive boundary arithmetic is odd but internally consistent: the tested seam cases show that each ledger lands in exactly one batch, so I could not construct a real duplicate-row trigger.

### Lesson Learned

Suspicious-looking batch math needs a concrete counterexample before it becomes a finding. When the package already has targeted seam tests, use them to distinguish weird style from an actual data-corruption path.
