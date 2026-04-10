# H002: Transaction Parquet drops `tx_signers`

**Date**: 2026-04-10
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_transactions` should emit the same signer set in both JSON and Parquet outputs. For classic transactions and fee-bump transactions alike, downstream Parquet readers should be able to see the `tx_signers` array that `TransformTransaction()` derives from the decorated signatures.

## Mechanism

`TransformTransaction()` fills `TxSigners`, and the JSON schema exposes that field as `tx_signers`. The Parquet path never carries it across: `TransactionOutputParquet` has no `tx_signers` column and `TransactionOutput.ToParquet()` has no corresponding assignment, so Parquet exports silently drop signer data that exists in JSON.

## Trigger

Run `export_transactions --write-parquet` on any ledger range containing a transaction with one or more signatures, especially a fee-bump transaction whose signer set is operationally important. The JSON export will include `tx_signers`, while the Parquet file will have no equivalent column.

## Target Code

- `internal/transform/transaction.go:227-300` — transaction transform populates `TxSigners`
- `internal/transform/schema.go:41-84` — JSON schema includes `tx_signers`
- `internal/transform/schema_parquet.go:31-74` — Parquet schema has no `tx_signers` field
- `internal/transform/parquet_converter.go:59-103` — `ToParquet()` never copies signer data
- `cmd/export_transactions.go:63-65` — this command writes the lossy Parquet output

## Evidence

`getTxSigners()` is called in both classic and fee-bump branches, and the transformed struct assigns `TxSigners` before returning. The JSON schema retains the field, but the Parquet schema/converter omit it entirely, so the exported Parquet shape cannot represent the source value.

## Anti-Evidence

The JSON export is correct, so users consuming line-delimited JSON are unaffected. This is specifically a Parquet consistency bug.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformTransaction()` in `transaction.go` populates `TxSigners` at line 270 (classic path) and line 300 (fee-bump path) by calling `getTxSigners()` on the envelope signatures. The JSON schema (`TransactionOutput` in `schema.go:83`) includes `TxSigners []string` with json tag `tx_signers`, so JSON output is complete. However, `TransactionOutputParquet` in `schema_parquet.go:32-74` has no `TxSigners` field at all, and the `ToParquet()` converter at `parquet_converter.go:59-103` maps every other field but silently drops `TxSigners`. Any downstream consumer reading Parquet output will see no signer data.

### Code Paths Examined

- `internal/transform/transaction.go:227-300` — Confirmed `TxSigners` is populated in both classic (line 270) and fee-bump (line 295-300) branches via `getTxSigners()`
- `internal/transform/schema.go:83` — Confirmed `TxSigners []string \`json:"tx_signers"\`` is present in `TransactionOutput`
- `internal/transform/schema_parquet.go:32-74` — Confirmed `TransactionOutputParquet` has NO `TxSigners` field; all other fields from `TransactionOutput` are present
- `internal/transform/parquet_converter.go:59-103` — Confirmed `ToParquet()` maps 33 fields but omits `TxSigners`; the function returns `TransactionOutputParquet` which structurally cannot hold signer data
- `cmd/export_transactions.go:63-65` — Confirmed Parquet write path calls `WriteParquet(transformedTransaction, parquetPath, new(transform.TransactionOutputParquet))`

### Findings

The bug is a straightforward omission: when `TransactionOutputParquet` was defined, the `TxSigners` field was not included in the Parquet schema struct. Consequently, the `ToParquet()` converter has no target field to assign it to. This creates a silent data discrepancy where JSON exports contain full signer information but Parquet exports contain none.

The omission affects every transaction in every Parquet export — not just an edge case. Since every Stellar transaction has at least one signature, every row in the Parquet output is missing signer data. This is particularly impactful for fee-bump transactions where the signer set is updated at line 300 to reflect the fee-bump envelope signatures.

Note: `ExtraSigners` (a separate precondition field) IS correctly carried through to Parquet at `schema_parquet.go:59` and `parquet_converter.go:87`. Only the actual transaction signers (`TxSigners`) are dropped.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go` (append a new test)
- **Setup**: Use an existing transaction test fixture that produces a `TransactionOutput` with a non-empty `TxSigners` slice
- **Steps**: Call `TransformTransaction()` to get a `TransactionOutput`, then call `.ToParquet()` on the result and cast to `TransactionOutputParquet`
- **Assertion**: Verify that the `TransactionOutputParquet` struct has no `TxSigners` field (compile-time check) — or more practically, compare the JSON-marshalled output of both structs and show that `tx_signers` is present in JSON but absent from Parquet. A field-count comparison between `TransactionOutput` and `TransactionOutputParquet` would also demonstrate the mismatch.
