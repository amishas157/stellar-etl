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

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The protocol-20 fee-bump workaround at `transaction.go:215-217` sets `outputFeeCharged = outputResourceFee - outputResourceFeeRefund`, omitting `outputInclusionFeeCharged`. The upstream authoritative `stellar/go` processor at `processors/transaction/transaction.go:234` uses the correct formula: `outputFeeCharged = outputResourceFee - outputResourceFeeRefund + outputInclusionFeeCharged`. This is a confirmed divergence from the upstream source of truth — the local code drops the inclusion fee term, underreporting `fee_charged` for every protocol-20 fee-bump Soroban transaction.

### Code Paths Examined

- `internal/transform/transaction.go:42` — initial `outputFeeCharged = int64(transaction.Result.Result.FeeCharged)` from XDR (known incorrect for P20 fee-bump)
- `internal/transform/transaction.go:162` — `outputResourceFee = int64(sorobanData.ResourceFee)`
- `internal/transform/transaction.go:177-179` — `initialFeeCharged = accountBalanceStart - accountBalanceEnd; outputInclusionFeeCharged = initialFeeCharged - outputResourceFee` — correctly computes inclusion fee
- `internal/transform/transaction.go:215-217` — **BUG**: `outputFeeCharged = outputResourceFee - outputResourceFeeRefund` — drops `outputInclusionFeeCharged`
- `stellar/go processors/transaction/transaction.go:233-234` — **upstream correct**: `outputFeeCharged = outputResourceFee - outputResourceFeeRefund + outputInclusionFeeCharged`
- `internal/transform/transaction_test.go:650` — only test uses `LedgerVersion: 25`, so P20 workaround path has zero test coverage

### Findings

The local stellar-etl diverges from the upstream `stellar/go` transaction processor on this exact line. The upstream formula is:
```go
outputFeeCharged = outputResourceFee - outputResourceFeeRefund + outputInclusionFeeCharged
```
The local formula is:
```go
outputFeeCharged = outputResourceFee - outputResourceFeeRefund
```
The missing `+ outputInclusionFeeCharged` term means every P20 fee-bump Soroban transaction's `fee_charged` export is too low by exactly the inclusion fee amount. This is a financial data corruption bug affecting historical ledger exports for ledger versions 20 through (but not including) 21.

The fix is a one-line change: add `+ outputInclusionFeeCharged` to line 216.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go`
- **Setup**: Create a test case with a fee-bump Soroban transaction where `LedgerVersion = 20`. Set up FeeChanges that produce a known `initialFeeCharged` (e.g., account balance drops from 10000 to 8000 → initialFeeCharged = 2000). Set `sorobanData.ResourceFee = 1500` so `inclusionFeeCharged = 2000 - 1500 = 500`. Set TxChangesAfter to produce `resourceFeeRefund = 200`.
- **Steps**: Call `TransformTransaction()` with the constructed input. Extract `FeeCharged` and `InclusionFeeCharged` from the output.
- **Assertion**: Assert `output.FeeCharged == 1500 - 200 + 500` (= 1800, the correct value). The current buggy code will produce `output.FeeCharged == 1500 - 200` (= 1300), which is 500 less than the correct value — exactly the missing inclusion fee.
