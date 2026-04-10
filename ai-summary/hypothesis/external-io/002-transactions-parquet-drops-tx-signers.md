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
