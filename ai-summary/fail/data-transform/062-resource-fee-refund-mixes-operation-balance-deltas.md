# H062: `resource_fee_refund` mixes in operation-level fee-account balance changes

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_transactions.resource_fee_refund` should reflect only the post-apply
fee refund credited back to the fee account. It should not include unrelated
balance changes caused by the transaction's actual operations.

## Mechanism

`TransformTransaction()` computes the refund by scanning
`meta.TxChangesAfter` / `metav4.TxChangesAfter + PostTxApplyFeeChanges` for the
fee account and subtracting the first seen balance from the last seen balance.
That looked like it might accidentally net together unrelated balance deltas if
the fee account also changed for another reason during the transaction.

## Trigger

Run `export_transactions` on a Soroban transaction where the fee account also
has a separate balance mutation from transaction execution, not just fee
processing, and compare the exported `resource_fee_refund` against the pure
post-apply refund amount.

## Target Code

- `internal/transform/transaction.go:177-202` — refund calculation scans whole change slices for the fee account
- `internal/transform/transaction.go:306-333` — helper reduces the slice to a single start/end balance pair

## Evidence

The helper ignores change provenance and only remembers one starting balance and
one ending balance for the fee account, so a mixed change slice would indeed
collapse distinct causes into a single delta.

## Anti-Evidence

The XDR meta layout prevents the mixed slice in the first place. `TxChangesAfter`
is explicitly the transaction-level change bucket that happens after operations,
while operation-caused balance mutations live under the per-operation metadata
arrays. The refund helper is therefore operating on the correct post-apply fee
change bucket rather than a bag of all account mutations.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected corruption would require operation-level account balance changes to
appear inside `TxChangesAfter`, but the XDR meta schema separates those changes
from the tx-level post-apply bucket that this helper reads. The code is coarse,
but it is reading the right bucket.

### Lesson Learned

Before treating a "scan the whole change slice" pattern as a delta-mixing bug,
check whether the upstream metadata schema has already partitioned the slice by
phase. A blunt reducer can still be correct when the XDR buckets are phase-pure.
