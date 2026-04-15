# H001: `history_transactions` drops protocol-23+ `PostTxApplyFeeChanges`

**Date**: 2026-04-15
**Subsystem**: data-transform
**Severity**: High
**Impact**: raw transaction export omits fee-refund XDR needed to explain exported refund fields
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a protocol-23+ Soroban transaction whose refund leg is carried in `LedgerTransaction.PostTxApplyFeeChanges`, the `history_transactions` row should preserve the full fee lifecycle in its raw XDR export. A downstream consumer that reads the row's raw blob fields should be able to recover the same fee-refund mutations that produced `resource_fee_refund`, either because an existing blob includes them or because the row exports a dedicated post-apply fee blob.

## Mechanism

`TransformTransaction()` already knows that protocol-23+ refunds moved into `transaction.PostTxApplyFeeChanges`: it appends that slice when computing `resource_fee_refund`. But the same function still base64-encodes only `transaction.UnsafeMeta` and `transaction.FeeChanges`, and `TransactionOutput` has no field where `PostTxApplyFeeChanges` can be emitted. The exported row therefore contains refund-derived numeric fields whose underlying raw XDR leg is silently absent from every `history_transactions` column.

## Trigger

Export a fee-bump Soroban transaction with transaction-meta V4 and non-empty `PostTxApplyFeeChanges`, like the existing fixture in `makeTransactionTestInput()` test case 4. The transformed row will contain a non-zero `resource_fee_refund`, but `tx_meta` round-trips only `UnsafeMeta`, `tx_fee_meta` round-trips only pre-apply `FeeChanges`, and no output field preserves `MarshalBase64(transaction.PostTxApplyFeeChanges)`.

## Target Code

- `internal/transform/transaction.go:59-67` — marshals `tx_meta` and `tx_fee_meta` but never serializes `PostTxApplyFeeChanges`
- `internal/transform/transaction.go:195-202` — computes `resource_fee_refund` from `append(metav4.TxChangesAfter, transaction.PostTxApplyFeeChanges...)`
- `internal/transform/schema.go:50-53` — `TransactionOutput` exposes only `tx_meta` and `tx_fee_meta`, with no post-apply fee blob field

## Evidence

The current transaction fixture already exercises this exact protocol-23+ shape: `transaction_test.go` test case 4 sets non-empty pre-apply `FeeChanges` and non-empty `PostTxApplyFeeChanges`, with a documented refund of `10,705` stroops (`internal/transform/transaction_test.go:572-625`). In production code, `TransformTransaction()` consumes that post-apply slice for refund accounting but discards it when building the raw XDR columns, so the output row is internally incomplete even though the input object contains the missing refund leg.

## Anti-Evidence

Prior review notes correctly establish that `tx_fee_meta` itself is historically the pre-apply deduction blob, so the fix is not to reinterpret that existing column. Horizon-style compatibility may therefore explain why `tx_fee_meta` stayed deduction-only, but it does not explain why `history_transactions` exposes no separate field for the protocol-23+ refund stream that the same transform now depends on numerically.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of success/data-transform/023-ledger-transaction-drops-post-tx-apply-fee-xdr
**Failed At**: reviewer

### Trace Summary

Traced `TransformTransaction()` in `internal/transform/transaction.go` lines 59-67 (XDR marshaling) and 195-211 (P23+ refund computation). Confirmed the function marshals only `transaction.UnsafeMeta` and `transaction.FeeChanges` into output blobs while using `transaction.PostTxApplyFeeChanges` for numeric refund computation without serializing it. The `TransactionOutput` schema in `schema.go:41-84` has no field for post-apply fee changes XDR.

### Code Paths Examined

- `internal/transform/transaction.go:59-67` — Confirmed: marshals only `UnsafeMeta` → `TxMeta` and `FeeChanges` → `TxFeeMeta`, no `PostTxApplyFeeChanges` serialization
- `internal/transform/transaction.go:195-211` — Confirmed: uses `append(metav4.TxChangesAfter, transaction.PostTxApplyFeeChanges...)` for refund calculation
- `internal/transform/schema.go:41-84` — Confirmed: `TransactionOutput` has `TxMeta` and `TxFeeMeta` but no post-apply fee changes blob field

### Why It Failed

This is a duplicate of the already-confirmed success finding `023-ledger-transaction-drops-post-tx-apply-fee-xdr`. Success #023 identified the identical root cause: `PostTxApplyFeeChanges` is not preserved in any raw XDR output field because neither `TransactionOutput` nor `LedgerTransactionOutput` has a field for it. Success #023 explicitly references `internal/transform/transaction.go:TransformTransaction:195-200` as affected code showing the same pattern. The mechanism, root cause, and fix direction (add a dedicated output field for `PostTxApplyFeeChanges`) are identical across both export paths.

### Lesson Learned

When a raw-XDR-omission finding is confirmed for one export path (e.g., `ledger_transaction`), check whether the parallel export path (`history_transactions`) was already covered by the same finding before filing a separate hypothesis. Success #023's suggested fix ("Add a dedicated raw output field for PostTxApplyFeeChanges") applies equally to both output schemas.
