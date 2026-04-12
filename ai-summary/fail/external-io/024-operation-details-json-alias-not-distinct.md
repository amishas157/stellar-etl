# H024: Operation Parquet looked like it dropped `details_json`, but that field is only a JSON alias of `details`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: missing operation-detail field
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `details_json` were a distinct operation payload, Parquet would need to preserve it separately from `details` so downstream consumers could recover both exported values. A true bug here would require `details_json` to differ semantically from the existing `details` field.

## Mechanism

I investigated whether `OperationOutputParquet` was silently dropping a second operation-details field by omitting `OperationDetailsJSON` from the schema and converter. That would only be a real corruption bug if the JSON export produced different values for `details` and `details_json`.

## Trigger

Export operations with `export_operations --write-parquet` on any ledger range containing populated operation details, then compare JSON fields `details` and `details_json` against the single Parquet `details` column.

## Target Code

- `internal/transform/operation.go:TransformOperation:85-98` — assigns both `OperationDetails` and `OperationDetailsJSON` from the same `outputDetails` map
- `internal/transform/schema.go:OperationOutput:137-150` — exposes both JSON fields on the operation output struct
- `internal/transform/schema_parquet.go:OperationOutputParquet:116-129` — defines only one `details` Parquet column
- `internal/transform/parquet_converter.go:OperationOutput.ToParquet:147-160` — serializes only `OperationDetails`

## Evidence

At first glance, the schema mismatch is suspicious: JSON has both `details` and `details_json`, while Parquet has only `details`. That looked like a straightforward Parquet field drop.

## Anti-Evidence

`TransformOperation()` populates both struct fields from the exact same `outputDetails` map, so `details_json` is not a second independent value. The Parquet export still preserves the underlying operation-details payload through its single `details` column.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`details_json` is just an alias of `details` in the current codebase, not a distinct external-io field. Omitting the alias from Parquet does not change the actual serialized operation-details content.

### Lesson Learned

Before treating a JSON-only field as a Parquet omission bug, verify whether it carries independent semantics or merely duplicates another exported value. Alias fields can look like schema drift while still preserving identical data.
