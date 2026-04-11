# H002: Transaction parquet fabricates absent optional fields

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For transactions that do not carry optional muxed-account or fee-bump metadata, the Parquet export should preserve those columns as null / absent, matching the JSON output. In particular, classic non-fee-bump transactions should not acquire synthetic values for `account_muxed`, `fee_account`, `fee_account_muxed`, `inner_transaction_hash`, or `new_max_fee`.

## Mechanism

`TransformTransaction()` only populates these fields in the muxed-account and fee-bump branches, and the JSON schema marks them optional with `omitempty`. The Parquet schema instead defines them as required `string` / `int64` columns, and `ToParquet()` copies zero values directly, so classic rows are exported with fabricated `""` and `0` values that look like real data rather than missing data.

## Trigger

Export any classic transaction whose source account is not muxed and whose envelope is not fee-bump. The JSON row omits these fields, while the Parquet row serializes empty strings and `new_max_fee = 0`.

## Target Code

- `internal/transform/transaction.go:273-301` — only sets muxed-account and fee-bump fields inside conditional branches
- `internal/transform/schema.go:45,60-63` — JSON schema declares these fields optional with `omitempty`
- `internal/transform/schema_parquet.go:36,51-55` — Parquet schema forces concrete string / int64 columns
- `internal/transform/parquet_converter.go:64,79-83` — converter writes zero values into Parquet rows

## Evidence

The transaction test fixture's first classic transaction leaves all fee-bump-only fields unset, which matches the JSON contract. The Parquet path has no nullable wrapper for those same fields, so absent metadata is converted into concrete empty values during serialization.

## Anti-Evidence

An empty string is not a valid Stellar account or transaction hash, so downstream consumers can sometimes infer that the value is synthetic. But that still changes the output contract from "missing" to "present with a value," which breaks null-sensitive analytics and is the same class of corruption already proven elsewhere in the repository for other optional columns.
