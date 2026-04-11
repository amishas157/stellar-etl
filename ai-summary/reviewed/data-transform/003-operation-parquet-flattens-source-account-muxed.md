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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Success/011 covers the same pattern for effects `address_muxed` but targets a different entity, code path, and exported table.

### Trace Summary

`TransformOperation()` (operation.go:40-47) creates a `null.String` for muxed source accounts. For non-muxed accounts, this stays at its zero value `{Valid: false, String: ""}`. On line 87, `outputSourceAccountMuxed.String` extracts the empty string into `OperationOutput.SourceAccountMuxed` (a plain `string`). JSON serialization omits the field via `omitempty` (schema.go:139), but `ToParquet()` (parquet_converter.go:150) copies the empty string into `OperationOutputParquet.SourceAccountMuxed`, which is a required Parquet string column with no `repetitiontype=OPTIONAL` tag (schema_parquet.go:119).

### Code Paths Examined

- `internal/transform/operation.go:40-47` — `null.String` only populated for `CryptoKeyTypeKeyTypeMuxedEd25519`; non-muxed accounts leave it invalid (zero value)
- `internal/transform/operation.go:85-87` — `OperationOutput{SourceAccountMuxed: outputSourceAccountMuxed.String}` extracts `.String` from the `null.String`, producing `""` for non-muxed accounts
- `internal/transform/schema.go:139` — `SourceAccountMuxed string` with `json:"source_account_muxed,omitempty"` — JSON omits the zero-value field
- `internal/transform/schema_parquet.go:119` — `SourceAccountMuxed string` with `parquet:"name=source_account_muxed, type=BYTE_ARRAY, convertedtype=UTF8, encoding=PLAIN_DICTIONARY"` — no optional marker, so Parquet writes it as a required field
- `internal/transform/parquet_converter.go:150` — `SourceAccountMuxed: oo.SourceAccountMuxed` directly copies the empty string into the Parquet struct

### Findings

The bug is confirmed and follows the exact same mechanism as the published finding success/011 (effects `address_muxed`), but in the operations entity. The nullability information created by `null.String` in `TransformOperation()` is destroyed in two stages: first when `.String` is accessed to populate the plain `string` field in `OperationOutput`, and second when the Parquet converter writes the empty string as a concrete value into a required Parquet column.

This creates a JSON/Parquet semantic disagreement: JSON omits the field (correct), while Parquet writes `""` (incorrect — should be null). Any downstream SQL query using `WHERE source_account_muxed IS NULL` will get zero results when it should match all non-muxed operation rows. This affects the vast majority of operations on Stellar mainnet since muxed accounts are uncommon.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Use an existing non-muxed operation test fixture (any regular `G...` source account operation). Call `TransformOperation()` to get an `OperationOutput`.
- **Steps**: Call `.ToParquet()` on the resulting `OperationOutput` and cast to `OperationOutputParquet`. Inspect the `SourceAccountMuxed` field.
- **Assertion**: Assert that `SourceAccountMuxed` in the Parquet output is `""` (empty string), demonstrating that the null/absent semantic from the JSON path is lost. Compare with the JSON struct where `omitempty` would suppress the field.
