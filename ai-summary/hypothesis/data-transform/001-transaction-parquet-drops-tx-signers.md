# H001: Transaction Parquet export drops `tx_signers`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a transaction carries one or more signatures, the exported Parquet row for `history_transactions` should preserve the same `tx_signers` array that `TransformTransaction()` puts into the JSON `TransactionOutput`. A JSON row and its Parquet twin should agree on whether the transaction was signed by one signer or several.

## Mechanism

`TransformTransaction()` now populates `TransactionOutput.TxSigners`, and the JSON schema exposes `tx_signers`, but the Parquet schema and converter never added a matching column. As a result, Parquet exports silently drop signer data entirely while JSON exports still include it, so downstream analytics that rely on Parquet cannot reconstruct the transaction signer set.

## Trigger

Export any ledger containing a transaction with at least one signature using Parquet output for `history_transactions`. Compare the JSON row's populated `tx_signers` array with the Parquet schema/row: the Parquet export has no `tx_signers` column at all.

## Target Code

- `internal/transform/schema.go:TransactionOutput:41-84` - JSON schema includes `TxSigners []string`.
- `internal/transform/transaction.go:TransformTransaction:227-300` - transform populates `TxSigners` for both regular and fee-bump transactions.
- `internal/transform/schema_parquet.go:TransactionOutputParquet:32-74` - Parquet schema omits any `TxSigners` field.
- `internal/transform/parquet_converter.go:TransactionOutput.ToParquet:59-103` - converter never maps `TxSigners`.

## Evidence

`transaction_test.go` asserts populated `TxSigners` values in the JSON transform output, so signer extraction is part of the current transform contract. The omission is localized to the Parquet surface: neither `TransactionOutputParquet` nor `TransactionOutput.ToParquet()` has a place to carry the array.

## Anti-Evidence

The repository may have historical consumers that never expected signer data in Parquet because the column was added to JSON later. If Parquet exports for `history_transactions` are intentionally frozen to an older schema, this would be a compatibility decision rather than an accidental omission.
