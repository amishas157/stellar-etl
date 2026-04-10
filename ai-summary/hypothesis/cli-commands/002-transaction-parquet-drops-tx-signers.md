# H002: Transaction parquet export drops `tx_signers` entirely

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_transactions` is run with `--write-parquet`, the parquet rows should preserve the same `tx_signers` data that the JSON rows expose for the same transactions. Downstream consumers choosing parquet should not silently lose signer metadata that exists in the JSON export.

## Mechanism

`TransformTransaction` populates `TransactionOutput.TxSigners`, and the JSON path writes that struct directly. But `TransactionOutputParquet` has no `tx_signers` field, and `TransactionOutput.ToParquet()` never serializes it. As a result, `export_transactions --write-parquet` silently strips signer data from every parquet row while the JSON file from the same run still contains the field.

## Trigger

Run `export_transactions` with both JSON output and `--write-parquet` over any ledger range containing transactions with signatures. Compare a JSON row and its corresponding parquet row: the JSON row contains `tx_signers`, while the parquet schema has no such column.

## Target Code

- `internal/transform/schema.go:41-84` — JSON transaction schema includes `tx_signers`
- `internal/transform/transaction.go:227-230,270-300` — transform populates `TxSigners`
- `internal/transform/schema_parquet.go:32-74` — parquet transaction schema omits `tx_signers`
- `internal/transform/parquet_converter.go:59-102` — transaction parquet conversion does not serialize `TxSigners`
- `cmd/export_transactions.go:51-65` — command appends transformed transactions and writes parquet from the truncated parquet schema

## Evidence

The JSON schema and transform path clearly carry `TxSigners`, but the parquet schema stops at `rent_fee_charged`. Because `WriteParquet` serializes `record.ToParquet()` with `new(transform.TransactionOutputParquet)`, there is no path for `tx_signers` to reach parquet output.

## Anti-Evidence

If a downstream dataset intentionally excludes signer metadata from parquet, the omission could be framed as a schema decision rather than a bug. But the command exposes JSON and parquet as alternate representations of the same export, and the missing parquet column is currently undocumented and silent.
