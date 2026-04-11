# H003: Operation parquet flattens absent source_account_muxed to empty string

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an operation source account is a regular `G...` account instead of a muxed `M...` account, `source_account_muxed` should remain null / absent in Parquet just as it is omitted from the JSON export. The export should distinguish "no muxed source account" from "present but empty."

## Mechanism

`TransformOperation()` only fills `SourceAccountMuxed` when the source account type is `KEY_TYPE_MUXED_ED25519`; otherwise the field stays at its zero value and JSON omits it. `OperationOutputParquet` models the column as a required string and `ToParquet()` copies the zero value directly, so every non-muxed operation is serialized with `source_account_muxed = ""`.

## Trigger

Export any operation whose effective source account is a normal ed25519 account rather than a muxed account. Compare the JSON output, which omits `source_account_muxed`, against the Parquet row, which contains an empty string in that column.

## Target Code

- `internal/transform/operation.go:40-47` — only populates muxed source metadata for muxed accounts
- `internal/transform/operation.go:85-88` — stores the empty-string default in `OperationOutput`
- `internal/transform/schema.go:138-149` — JSON schema marks `source_account_muxed` optional
- `internal/transform/schema_parquet.go:118-129` — Parquet schema requires a concrete string column
- `internal/transform/parquet_converter.go:147-150` — converter serializes the zero-value string into Parquet

## Evidence

The transform path uses a local `null.String` for muxed metadata but discards that nullability by assigning `.String` into `OperationOutput`. That already erases the distinction at the struct boundary, and the Parquet schema then persists the fabricated empty string instead of a null.

## Anti-Evidence

Downstream consumers can sometimes treat `""` as "missing" because no valid muxed address is empty. But the exported value is still wrong: JSON and Parquet no longer agree on field presence, and SQL/BigQuery queries that rely on `IS NULL` semantics will misclassify normal operations.
