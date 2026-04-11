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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `TransformTransaction()` (transaction.go:20–303) end-to-end, focusing on the `tx_fee_meta` serialization at line 64 and the P23+ refund computation at lines 195–211. Confirmed that `tx_fee_meta` has always been the base64-serialized `transaction.FeeChanges` — the pre-apply fee debit — and has never included post-apply refund changes. The test suite's P23+ fixture (test case 4, transaction_test.go:178–217) explicitly asserts this convention with a populated `PostTxApplyFeeChanges` alongside a `TxFeeMeta` that encodes only `FeeChanges`.

### Code Paths Examined

- `internal/transform/transaction.go:64` — `outputTxFeeMeta, err := xdr.MarshalBase64(transaction.FeeChanges)` — marshals only pre-apply fee changes, consistent with the historical field contract
- `internal/transform/transaction.go:195-211` — P23+ V4 meta path correctly appends `PostTxApplyFeeChanges` to compute `outputResourceFeeRefund`, but this is for the numeric column, not the blob
- `internal/transform/transaction.go:245` — `TxFeeMeta: outputTxFeeMeta` — assigns the pre-apply-only blob
- `internal/transform/transaction_test.go:178-217` — Test case 4 has `PostTxApplyFeeChanges` populated with a 10,705-stroop refund yet asserts `TxFeeMeta` contains only the `FeeChanges` serialization (debit of 38,533). This is an explicit design-intent assertion.
- `internal/transform/transaction_test.go:572-620` — The test fixture's `FeeChanges` (balance 4,067,134,559,286 → 4,067,134,520,753) and `PostTxApplyFeeChanges` (balance 4,067,134,520,753 → 4,067,134,531,458) are separate slices, and the test deliberately validates that only `FeeChanges` populates `TxFeeMeta`.
- `internal/transform/ledger_transaction.go:32` — The sibling `TransformLedgerTransaction()` uses the same `transaction.FeeChanges`-only convention for its `tx_fee_meta`.

### Why It Failed

The hypothesis's expected behavior is incorrect. `tx_fee_meta` has **never** contained refund changes — across any protocol version. Its contract is to serialize `transaction.FeeChanges`, which represents the pre-transaction-apply fee debit. Before P23, the refund was available in `tx_meta` (via `TxChangesAfter` in `TransactionMetaV3`), not in `tx_fee_meta`. The P23+ test fixture (test case 4) was written with full knowledge of `PostTxApplyFeeChanges` and intentionally asserts that `TxFeeMeta` encodes only the debit-side `FeeChanges`. This is working-as-designed behavior matching the Horizon ecosystem convention where `fee_changes_xdr` is the pre-apply fee processing changes.

### Lesson Learned

`tx_fee_meta` is the pre-transaction-apply fee debit blob, not the complete fee lifecycle. Before P23, the refund was recoverable from `tx_meta` (within `TxChangesAfter`). The fact that P23+ moved refunds out of `TransactionMeta` into `PostTxApplyFeeChanges` (which is not serialized into any blob) is a data-completeness concern at the schema level — but it does not make `tx_fee_meta`'s existing contract wrong. A hypothesis about missing refund XDR should target the absence of a dedicated `PostTxApplyFeeChanges` blob column, not the pre-existing `tx_fee_meta` field.
