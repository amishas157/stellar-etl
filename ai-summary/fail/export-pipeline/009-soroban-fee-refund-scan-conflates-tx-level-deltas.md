# H009: Soroban refund accounting conflates unrelated tx-level account deltas

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`resource_fee_refund` should be derived only from the fee-account refund delta, not from unrelated account mutations that happen elsewhere in transaction metadata. If the fee-paying account appears multiple times in metadata for independent reasons, the export should isolate the refund-specific balance change.

## Mechanism

`TransformTransaction()` computes both `inclusion_fee_charged` and `resource_fee_refund` by scanning `FeeChanges`, `TxChangesAfter`, and `PostTxApplyFeeChanges` with `getAccountBalanceFromLedgerEntryChanges()`. Because that helper just overwrites `accountBalanceStart` / `accountBalanceEnd` for any matching account row it sees, it initially looked like a transaction that touched the fee account for non-fee reasons could silently corrupt the exported refund math.

## Trigger

Process a Soroban transaction where the fee-paying account appears multiple times in `TxChangesAfter` or `PostTxApplyFeeChanges` for reasons other than fee charging/refunding, then compare the exported `resource_fee_refund` against the isolated refund delta.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:177-202` — derives refund/inclusion fee from whole-slice balance scans
- `internal/transform/transaction.go:getAccountBalanceFromLedgerEntryChanges:306-333` — keeps only the last matching start/end pair for a fee account

## Evidence

The helper does not distinguish fee-processing balance changes from any other account-entry change; it matches only on account ID and change type. On first read, that makes the refund math look vulnerable to any additional tx-level balance mutation for the same fee account.

## Anti-Evidence

Upstream `TransactionMetaV3` / `TransactionMetaV4` define `TxChangesAfter` as transaction-level changes after operations, while protocol-23+ refund changes move into `PostTxApplyFeeChanges`. In current SDK contracts and repository fixtures, operation-level balance effects live in operation metadata, and the fee-account slices used here only carry fee/refund bookkeeping.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I did not find a concrete live metadata path where unrelated fee-account balance changes are stored in the same `TxChangesAfter` / `PostTxApplyFeeChanges` collections that this code scans. The current XDR and ingest contracts segregate transaction-level fee bookkeeping from operation-level balance effects closely enough that the suspected conflation lacks a realistic trigger.

### Lesson Learned

When Soroban fee math looks suspicious, check the metadata-layer contract first: `TxChangesAfter` and `PostTxApplyFeeChanges` are specialized transaction-level slices, not generic dumps of all operation balance changes. Without a concrete XDR path that mixes unrelated fee-account deltas into those slices, the overwrite pattern alone is not enough to claim corruption.
