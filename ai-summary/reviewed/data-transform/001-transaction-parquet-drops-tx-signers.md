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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransactionOutput` (schema.go:83) defines `TxSigners []string` with a JSON tag. `TransformTransaction()` (transaction.go:227-300) populates this field via `getTxSigners()` for both regular and fee-bump transactions. However, `TransactionOutputParquet` (schema_parquet.go:32-74) has no corresponding field, and `TransactionOutput.ToParquet()` (parquet_converter.go:59-102) never references `TxSigners`. The field is silently dropped during Parquet conversion. This is a distinct finding from the already-published success/001 (which addresses wrong *content* in `tx_signers`); this hypothesis addresses the complete *absence* of the column from Parquet output.

### Code Paths Examined

- `internal/transform/schema.go:83` — `TxSigners []string` is the last field in `TransactionOutput`, confirming it exists in the JSON schema
- `internal/transform/schema_parquet.go:32-74` — `TransactionOutputParquet` has 42 fields; none is `TxSigners`. The struct ends at `RentFeeCharged` (line 73)
- `internal/transform/parquet_converter.go:59-102` — `ToParquet()` maps 40 fields from `TransactionOutput` to `TransactionOutputParquet`; `TxSigners` is not among them
- `internal/transform/transaction.go:227,270` — `getTxSigners()` is called and result assigned to `TxSigners` for regular transactions
- `internal/transform/transaction.go:295,300` — `getTxSigners()` called and `TxSigners` reassigned for fee-bump transactions
- `internal/transform/transaction.go:349` — `getTxSigners()` iterates `[]xdr.DecoratedSignature` and encodes each

### Findings

The structural gap is confirmed: `TxSigners` is populated in every `TransactionOutput` produced by `TransformTransaction()` and serialized to JSON, but completely absent from the Parquet output path. The Parquet struct has no field for it, and the converter has no mapping for it. Any downstream consumer reading Parquet `history_transactions` data cannot access signer information at all.

Note: The already-published finding `success/data-transform/001-tx-signers-encode-signature-bytes` documents that the *values* in `TxSigners` are wrong (signature bytes encoded as fake account IDs). This hypothesis is independent — it documents the complete absence of the column from Parquet regardless of what the values contain. Even after the content bug is fixed, Parquet consumers would still lack the field unless the Parquet schema and converter are also updated.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or `internal/transform/transaction_test.go` if no dedicated converter test exists)
- **Setup**: Use an existing `TransactionOutput` fixture that has a non-empty `TxSigners` slice (e.g., from the test cases at `transaction_test.go:112,143,176,216`)
- **Steps**: Call `transactionOutput.ToParquet()` and type-assert the result to `TransactionOutputParquet`. Attempt to access a `TxSigners` field on the result.
- **Assertion**: Demonstrate that `TransactionOutputParquet` has no `TxSigners` field (compile-time absence), confirming the column is structurally missing from Parquet output. Alternatively, serialize the Parquet struct and show the column is absent from the schema.
