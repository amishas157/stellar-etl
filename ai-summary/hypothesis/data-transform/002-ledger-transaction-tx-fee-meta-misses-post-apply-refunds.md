# H002: `ledger_transaction.tx_fee_meta` drops P23+ fee-refund changes

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: financial reconciliation metadata omits refund ledger changes
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For protocol-23+ Soroban transactions, the `ledger_transaction` export should preserve the full fee-processing XDR for the transaction, including any refund balance changes applied after execution. Consumers using `tx_fee_meta` from the raw ledger-transaction table should be able to replay the same fee-account mutation that the transaction actually experienced on-chain.

## Mechanism

`TransformLedgerTransaction()` serializes `tx_fee_meta` from `transaction.FeeChanges` and never incorporates `transaction.PostTxApplyFeeChanges`, even though P23+ moved refund credits into that second slice. As a result, the ledger-transaction table keeps exporting only the initial debit side of Soroban fee processing on P23+ ledgers, so the table silently loses the refund half of the fee lifecycle.

## Trigger

Run `export_ledger_transaction` over a protocol-23+ ledger containing a Soroban transaction with a non-zero refund. Decode the row's `tx_fee_meta` blob and compare it with the ledger's post-apply fee changes: the refund credit is absent because `TransformLedgerTransaction()` never serializes `PostTxApplyFeeChanges`.

## Target Code

- `internal/transform/ledger_transaction.go:27-35` — `TxFeeMeta` is marshaled from `transaction.FeeChanges`
- `internal/transform/ledger_transaction.go:47-53` — the incomplete blob is written into `LedgerTransactionOutput`
- `internal/transform/transaction.go:195-202` — sibling transaction export documents that P23+ refunds live in `PostTxApplyFeeChanges`

## Evidence

The ledger-transaction transform has not been updated alongside the P23+ refund fix in `TransformTransaction()`: it still treats `FeeChanges` as the entire fee-meta payload. The underlying ingest object already carries `PostTxApplyFeeChanges`, and the transaction transform's own protocol comment confirms those entries are the canonical location of P23+ refund changes.

## Anti-Evidence

There is no parallel numeric refund column in `LedgerTransactionOutput`, so this bug is easier to miss than the `history_transactions` mismatch and may have survived because the raw table has less direct test coverage. Reviewer confirmation is still needed on whether `ledger_transaction.tx_fee_meta` is meant to be a full fee-meta export or just the pre-apply fee-processing slice.
