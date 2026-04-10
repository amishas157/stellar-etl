# H001: Transaction parquet export drops `tx_signers`

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_transactions --write-parquet` serializes a transaction row, the parquet output should preserve the same `tx_signers` values that `TransformTransaction()` already places in JSON output. For the repository's existing transaction fixtures, rows that contain one fake `G...` signer string in `TransactionOutput.TxSigners` should produce a parquet row with that same signer list rather than silently omitting the column.

## Mechanism

`TransformTransaction()` always populates `TransactionOutput.TxSigners`, and the transaction tests assert non-empty `TxSigners` on multiple fixtures. But `TransactionOutputParquet` has no `tx_signers` field at all, and `TransactionOutput.ToParquet()` therefore has nowhere to copy the populated slice. The parquet export path silently strips signer data from every row even though the JSON export for the same command includes it.

## Trigger

Run `export_transactions` with `--write-parquet` on any normal signed transaction or fee-bump transaction. The JSON row contains `tx_signers`, but the parquet artifact has no `tx_signers` column/value for that row.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:227-300` — populates `TxSigners` for classic and fee-bump envelopes
- `internal/transform/schema.go:TransactionOutput:41-84` — JSON schema defines `TxSigners []string`
- `internal/transform/transaction_test.go:makeTransactionTestOutput:84-217` — existing expected outputs include non-empty `TxSigners`
- `internal/transform/schema_parquet.go:TransactionOutputParquet:31-74` — parquet schema omits `tx_signers`
- `internal/transform/parquet_converter.go:TransactionOutput.ToParquet:59-102` — parquet conversion never copies `TxSigners`
- `cmd/export_transactions.go:transactionsCmd.Run:51-65` — command writes parquet rows from `TransactionOutputParquet`

## Evidence

The codebase already supports repeated string fields in parquet via `ExtraSigners []string`, so the omission is not explained by a blanket "lists are unsupported" limitation. This looks like a schema/converter drift bug isolated to `tx_signers`.

## Anti-Evidence

The current `tx_signers` values are themselves known-bad identity data in JSON exports, so downstream users may already avoid the field. But the parquet path still corrupts the dataset shape further by removing the field entirely instead of matching the current JSON contract.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `TransformTransaction()` through `ToParquet()` to `WriteParquet()` in the export command. `TransactionOutput` defines `TxSigners []string` at schema.go:83, which is populated by `getTxSigners()` at transaction.go:227-270 (classic) and transaction.go:295-300 (fee-bump). However, `TransactionOutputParquet` (schema_parquet.go:31-74) has no corresponding field, and `ToParquet()` (parquet_converter.go:59-102) has no mapping for it. The export command at export_transactions.go:63-65 writes parquet via `WriteParquet()` using `TransactionOutputParquet` as the schema, so the column is silently absent.

### Code Paths Examined

- `internal/transform/schema.go:83` — confirms `TxSigners []string` field exists in JSON output struct
- `internal/transform/schema_parquet.go:31-74` — confirms NO `TxSigners` field in parquet struct; `ExtraSigners []string` IS present at line 59, proving `[]string` parquet support exists
- `internal/transform/parquet_converter.go:59-102` — confirms `ToParquet()` maps all fields EXCEPT `TxSigners`; `ExtraSigners` is mapped at line 87
- `internal/transform/transaction.go:227-270` — `getTxSigners()` called and result assigned to `TxSigners` for classic envelopes
- `internal/transform/transaction.go:295-300` — fee-bump path overwrites `TxSigners` with fee-bump signatures
- `internal/transform/transaction.go:349-360` — `getTxSigners()` encodes signature bytes as `VersionByteAccountID`
- `internal/transform/transaction_test.go:112,143,176,216` — four test fixtures assert non-empty `TxSigners`
- `cmd/export_transactions.go:51-65` — parquet write path uses `TransactionOutputParquet` schema

### Findings

The schema drift is confirmed: `TransactionOutputParquet` is missing the `TxSigners` field that `TransactionOutput` defines. The `ToParquet()` converter has no mapping for it. The parquet library uses the struct definition to determine columns, so the column is entirely absent from parquet output.

This is a genuine schema inconsistency between JSON and Parquet export formats. The pattern is identical to how `ExtraSigners` is handled (schema_parquet.go:59, parquet_converter.go:87), so adding the field follows an established pattern.

Severity downgraded from High to Medium because the confirmed success finding `001-tx-signers-encode-signatures` establishes that the values in `TxSigners` are already wrong (signature bytes encoded as fake account IDs). The parquet column is missing, but the data it would contain is itself incorrect. The practical impact is a schema contract mismatch between output formats, not new data corruption. A complete fix requires both adding the parquet column AND fixing the underlying value computation.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or create if not present; alternatively append to `internal/transform/transaction_test.go`)
- **Setup**: Construct a `TransactionOutput` with `TxSigners: []string{"GABC..."}` and call `ToParquet()`
- **Steps**: Cast the result to `TransactionOutputParquet` and inspect whether a `TxSigners` field exists via reflection, or simply verify the struct definition lacks the field
- **Assertion**: Assert that `TransactionOutputParquet` should contain a `TxSigners` field — currently it does not, confirming the schema drift. A compile-time check: `_ = TransactionOutputParquet{}.TxSigners` would fail to compile.
