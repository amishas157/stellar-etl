# H001: `history_transactions.tx_fee_meta` drops P23+ fee-refund changes

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: financial reconciliation metadata omits refund ledger changes
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Soroban transactions on protocol 23+ that receive a resource-fee refund, the exported `history_transactions.tx_fee_meta` field should encode the complete fee-processing ledger delta for that transaction. A downstream consumer decoding `tx_fee_meta` should see both the initial fee debit and the post-apply refund credit, so the raw XDR agrees with `resource_fee_refund`, `inclusion_fee_charged`, and the final fee-account balance change.

## Mechanism

`TransformTransaction()` still marshals `tx_fee_meta` from `transaction.FeeChanges` alone, even though the same function now knows that protocol-23+ refunds moved out of transaction meta and into `transaction.PostTxApplyFeeChanges`. The numeric fee columns were updated to append `PostTxApplyFeeChanges` when computing `resource_fee_refund`, but the raw `tx_fee_meta` payload was left on the pre-P23 path, so P23+ Soroban rows export a one-sided fee debit in `tx_fee_meta` while sibling columns claim a refund happened.

## Trigger

Export any ledger containing a protocol-23+ Soroban transaction with a non-zero resource-fee refund. Decode `history_transactions.tx_fee_meta` and compare it with the fee-account deltas implied by `resource_fee_refund`: the refund credit only appears in `PostTxApplyFeeChanges`, not in the exported `tx_fee_meta` blob.

## Target Code

- `internal/transform/transaction.go:64-67` — marshals `outputTxFeeMeta` from `transaction.FeeChanges`
- `internal/transform/transaction.go:195-202` — P23+ refund accounting explicitly appends `transaction.PostTxApplyFeeChanges`
- `internal/transform/transaction.go:232-245` — final transaction row persists the stale `outputTxFeeMeta`

## Evidence

`TransformTransaction()` contains an explicit protocol-23 comment explaining that fee refunds moved from `TxChangesAfter` to `PostTxApplyFeeChanges`, and it appends both slices when computing `outputResourceFeeRefund`. Despite that, `outputTxFeeMeta` is serialized earlier from `transaction.FeeChanges` only and never recomputed. The transaction tests already include a P23+ Soroban fee-bump fixture with populated `PostTxApplyFeeChanges`, which means the code path is live today rather than theoretical.

## Anti-Evidence

Before protocol 23, `transaction.FeeChanges` was sufficient for `tx_fee_meta`, so older ledgers would still look correct. The current tests also assert the existing `tx_fee_meta` value, so reviewer confirmation is still needed on whether the field is intended to remain a partial pre-refund snapshot or the full fee-processing metadata.
