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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/external-io/002-transactions-parquet-drops-tx-signers.md.gh-published`
**Failed At**: reviewer

### Trace Summary

The hypothesis correctly identifies that `TransactionOutputParquet` (schema_parquet.go:32-74) has no `TxSigners` field while `TransactionOutput` (schema.go:83) does. The `ToParquet()` converter (parquet_converter.go:59-102) maps every field except `TxSigners`, confirming the parquet export silently drops signer data. However, this exact finding has already been confirmed, PoC-verified, and published.

### Code Paths Examined

- `internal/transform/schema.go:83` — `TxSigners []string` present in JSON schema
- `internal/transform/schema_parquet.go:32-74` — `TransactionOutputParquet` ends at `RentFeeCharged`, no `TxSigners` field
- `internal/transform/parquet_converter.go:59-102` — `ToParquet()` copies all fields except `TxSigners`

### Why It Failed

This is a duplicate of an already-confirmed and published finding at `ai-summary/success/external-io/002-transactions-parquet-drops-tx-signers.md.gh-published`. That finding covers the identical root cause (missing `TxSigners` field in `TransactionOutputParquet` and `ToParquet()`), the same affected code paths, and includes a passing PoC test. The only difference is subsystem scoping — the prior finding was filed under `external-io` while this one targets `cli-commands`, but the bug resides in `internal/transform/` schema and converter code common to both.

### Lesson Learned

Parquet schema omission findings should be checked across all subsystem success directories, not just the hypothesis's own subsystem. The `tx_signers` parquet drop was already captured under `external-io` even though it equally affects the `cli-commands` export path.
