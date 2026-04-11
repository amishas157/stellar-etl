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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/fail/data-transform/summary.md` entry 012 ("Operation Parquet omits `details_json`")
**Failed At**: reviewer

### Trace Summary

Traced `TransformOperation` at `operation.go:85-98` and confirmed that both `OperationDetails` (line 92) and `OperationDetailsJSON` (line 97) are assigned the exact same variable `outputDetails`. They are literally the same Go map. The Parquet converter at `parquet_converter.go:153` serializes `oo.OperationDetails` via `toJSONString()`, producing a JSON string that contains the identical data. There is no information loss — `details_json` is just an alias for the same map, and the Parquet `details` column already encodes it fully.

### Code Paths Examined

- `internal/transform/operation.go:92,97` — both `OperationDetails` and `OperationDetailsJSON` assigned from same `outputDetails` variable
- `internal/transform/parquet_converter.go:153` — `OperationDetails: toJSONString(oo.OperationDetails)` serializes the map to a JSON string
- `internal/transform/schema.go:142,149` — JSON schema has both `details` (map) and `details_json` (map)
- `internal/transform/schema_parquet.go:122` — Parquet schema has `details` (string)

### Why It Failed

Two independent reasons:

1. **Duplicate**: This exact hypothesis was already investigated and rejected as `data-transform` fail entry 012. The summary documents: "Operation Parquet omits `details_json` — `TransformOperation()` writes the same map to both `OperationDetails` and `OperationDetailsJSON`; Parquet preserves it via the `details` column."

2. **No data loss**: `OperationDetails` and `OperationDetailsJSON` are assigned the same `outputDetails` map (operation.go lines 92 and 97). The Parquet `details` column serializes this map to a JSON string via `toJSONString()`. A downstream consumer can parse the JSON string to recover the exact same key-value pairs that `details_json` would have provided. The "missing" column is not a separate data source — it's an alias for data already present.

### Lesson Learned

When two schema fields are assigned from the same source variable, omitting one from Parquet is not data loss — verify assignment sources before claiming a column carries unique data. Also, check the data-transform fail summary for prior Parquet schema omission investigations.
