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
