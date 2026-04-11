# H003: Operation parquet export drops the `details_json` companion column

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_operations` is run with `--write-parquet`, the parquet rows should preserve the same operation payload columns that the JSON rows expose for the same transformed `OperationOutput`, including `details_json`. Downstream consumers choosing parquet should not silently lose a column present in the JSON export of the same ledger range.

## Mechanism

`TransformOperation` populates both `OperationDetails` and `OperationDetailsJSON` from the decoded operation detail map, and the JSON path emits both fields. But `OperationOutputParquet` defines only a single `details` string column and `OperationOutput.ToParquet()` serializes only that field, so parquet drops `details_json` entirely and collapses the structured companion column into an untyped string-only representation.

## Trigger

Run `export_operations --write-parquet` over any ledger range containing operations. Compare the JSON output and the parquet schema for the same row: JSON includes both `details` and `details_json`, while parquet has no `details_json` column at all.

## Target Code

- `internal/transform/operation.go:54-100` — `TransformOperation` populates both `OperationDetails` and `OperationDetailsJSON`
- `internal/transform/schema.go:136-149` — `OperationOutput` includes `details_json`
- `internal/transform/schema_parquet.go:116-129` — `OperationOutputParquet` omits `details_json`
- `internal/transform/parquet_converter.go:147-160` — `ToParquet()` serializes only `details`
- `cmd/export_operations.go:51-65` — command writes the same transformed operations to JSON and parquet

## Evidence

The JSON schema explicitly carries `OperationDetailsJSON map[string]interface{} \`json:"details_json"\``, and `TransformOperation` sets that field to `outputDetails` alongside `OperationDetails`. The parquet schema/converter pair never mention `OperationDetailsJSON`, so there is no code path by which `details_json` can survive into parquet output.

## Anti-Evidence

The parquet `details` string still contains similar payload information if a downstream reader reparses the JSON string manually. The deviation is the silent loss of the named `details_json` column and its typed companion representation, not the complete loss of operation details.
