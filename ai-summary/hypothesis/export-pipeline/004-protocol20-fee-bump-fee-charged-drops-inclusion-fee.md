# H004: Protocol-20 fee-bump Soroban `fee_charged` drops the inclusion fee

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For any protocol-20 Soroban fee-bump transaction, exported `fee_charged` should equal the full net fee deducted for the transaction: `inclusion_fee_charged + resource_fee - resource_fee_refund`. On a non-refundable fee-bump Soroban transaction, that means `fee_charged` should still include both the outer inclusion fee and the Soroban resource fee.

## Mechanism

`TransformTransaction()` contains a special protocol-20 fee-bump workaround, but the replacement formula is `resource_fee - resource_fee_refund`. That recomputation drops `inclusion_fee_charged` entirely, so every pre-protocol-21 fee-bump Soroban transaction underreports `fee_charged` by at least the inclusion-fee component when no refund occurs, and produces a malformed net-fee value even before any refund-recipient issues are considered.

## Trigger

1. Export a protocol-20 Soroban fee-bump transaction with `LedgerVersion < 21`.
2. Use a case with `resource_fee_refund = 0` (or any non-refundable Soroban fee-bump).
3. Inspect `history_transactions.fee_charged`, `inclusion_fee_charged`, and `resource_fee`.
4. The row will report `fee_charged = resource_fee` instead of `inclusion_fee_charged + resource_fee`.

## Target Code

- `internal/transform/transaction.go:177-180` — derives `inclusion_fee_charged` from the initial fee-account deduction.
- `internal/transform/transaction.go:215-217` — protocol-20 fee-bump workaround overwrites `fee_charged` with `resource_fee - resource_fee_refund`.
- `internal/transform/schema.go:37-44, 77-84` — transaction schema exports `fee_charged`, `inclusion_fee_charged`, `resource_fee`, and `resource_fee_refund` as independent numeric fields.

## Evidence

The function first computes `inclusion_fee_charged` from the pre-apply deduction (`initialFeeCharged - outputResourceFee`), establishing that the inclusion-fee component is available. It then overwrites `fee_charged` for all protocol-20 fee-bump Soroban transactions using only `resource_fee` and `resource_fee_refund`, which mathematically excludes the inclusion fee from the exported total.

## Anti-Evidence

The branch is guarded by an explicit comment referencing stellar-core issue 4188, so the code is clearly attempting to compensate for a real protocol-20 bug. A reviewer may conclude the intended contract was only to correct the refundable Soroban portion, but that still leaves the exported `fee_charged` inconsistent with the surrounding fee breakdown columns and with the net fee actually deducted.
