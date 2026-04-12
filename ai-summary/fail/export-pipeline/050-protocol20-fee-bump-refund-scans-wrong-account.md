# H005: Protocol-20 fee-bump Soroban `resource_fee_refund` scans the wrong account

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a protocol-20 fee-bump Soroban transaction affected by the pre-protocol-21 refund bug, exported `resource_fee_refund` should preserve the actual refunded amount visible in the ledger changes, even though stellar-core credited that refund to the inner transaction source instead of the fee-bump source. A refundable transaction should therefore export a positive `resource_fee_refund`, not `0`.

## Mechanism

`TransformTransaction()` always looks for refund deltas on `feeAccountAddress`. In protocol 20 fee-bump Soroban transactions, the refunded stroops were applied to the inner transaction source account instead of the outer fee-bump payer, so the exporter scans the wrong account in `TxChangesAfter` and silently leaves `resource_fee_refund` at zero even when the ledger metadata contains a real refund.

## Trigger

1. Export a protocol-20 Soroban fee-bump transaction with `LedgerVersion < 21` and a non-zero refundable resource fee.
2. Choose a case in the stellar-core-4188 class where the refund lands on the inner source account.
3. Inspect `history_transactions.resource_fee_refund`.
4. The row will export `0` even though the refund is present in `TransactionMetaV3.TxChangesAfter`.

## Target Code

- `internal/transform/transaction.go:154-159` — fee-bump path sets `feeAccountAddress` to the outer fee-bump source.
- `internal/transform/transaction.go:181-185` — V3 refund scan reads `TxChangesAfter` but matches only `feeAccountAddress`.
- `internal/transform/transaction.go:306-333` — balance-change helper only returns deltas for the one requested account address.

## Evidence

The protocol-20 workaround never switches the refund lookup away from the outer fee-bump account. Because the helper only returns deltas for the supplied address, any refund credited to the inner source account is invisible to the exporter and becomes `resource_fee_refund = 0` despite being present in `TxChangesAfter`.

## Anti-Evidence

The current code is correct for post-fix protocols where the refund belongs to the fee-paying account, and it is also correct for non-refundable transactions. This hypothesis depends on the specific protocol-20 fee-bump refund routing bug, so it only applies to that narrow but real historical transaction class.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full code path for fee-bump Soroban refund computation in `TransformTransaction()` (lines 151-217) and the helper `getAccountBalanceFromLedgerEntryChanges()` (lines 306-333). Then verified the actual nature of the stellar-core #4188 bug by reading the upstream issue. The issue is titled "FeeBumpTransactionFrame's feeCharged does not take refunds into account" and the description states: "This field [is] purely used for reporting purposes, so downstream consumers will display a fee larger than what was actually charged." The bug was in the **reported `feeCharged` field** of the `TransactionResult`, NOT in the refund routing. The refund was always applied to the correct account (the fee-bump payer); only the `feeCharged` reporting value was wrong.

### Code Paths Examined

- `internal/transform/transaction.go:151-159` — Sets `feeAccountAddress` to the fee-bump payer for fee-bump envelopes. This is the correct account for refund scanning because stellar-core #4188 did not change refund routing.
- `internal/transform/transaction.go:181-184` — V3 refund scan reads `TxChangesAfter` for `feeAccountAddress`. Since the refund IS on the fee-bump payer account (even in protocol 20), this correctly finds the refund balance change.
- `internal/transform/transaction.go:212-217` — Protocol-20 workaround recomputes `feeCharged` because `TransactionResult.FeeCharged` was wrong (too high, didn't subtract refund). Uses `outputResourceFeeRefund`, confirming the developers expected it to be a valid non-zero value.
- `internal/transform/transaction.go:306-333` — `getAccountBalanceFromLedgerEntryChanges()` scans for state/updated pairs on the requested account. Works correctly when the refund is on the fee-bump payer.
- https://github.com/stellar/stellar-core/issues/4188 — Upstream issue confirms the bug was "feeCharged does not take refunds into account" — a reporting field error, not a refund routing error.

### Why It Failed

The hypothesis incorrectly characterizes the stellar-core #4188 bug. The actual bug was that `FeeBumpTransactionFrame`'s `feeCharged` reporting field did not subtract refunds, causing it to report a higher fee than was actually charged. The refund itself was always correctly applied to the fee-bump payer account in `TxChangesAfter`. Therefore, the ETL's refund scan on `feeAccountAddress` finds the correct balance change, and `resource_fee_refund` is computed correctly for protocol-20 fee-bump transactions. The ETL's existing workaround (line 216) also corroborates this: it uses `outputResourceFeeRefund` in its formula, which would be meaningless if the scan always returned 0.

### Lesson Learned

When a hypothesis references an upstream bug (like stellar-core #4188), verify the actual nature of the upstream bug before assuming a specific mechanism. The issue title "feeCharged does not take refunds into account" clearly indicates a reporting field error, not a ledger state routing error. The ETL workaround formula `outputResourceFee - outputResourceFeeRefund` further confirms the refund scan is expected to produce a valid value for these transactions.
