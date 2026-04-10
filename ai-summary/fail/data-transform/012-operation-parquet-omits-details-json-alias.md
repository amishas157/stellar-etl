# H012: Operation Parquet omits `details_json`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: Low
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `history_operations.details_json` is meant to carry information distinct from `details`, the Parquet export should preserve both columns or otherwise document why one representation is intentionally omitted. A format change should not silently drop a second operation-details payload if it contains unique content.

## Mechanism

`OperationOutput` exposes both `OperationDetails` and `OperationDetailsJSON`, but `OperationOutputParquet` only serializes a single `details` string. At first glance this looks like another JSON-to-Parquet omission bug that could strip an operation-details column from Parquet exports.

## Trigger

Run `export_operations --write-parquet` and compare the JSON row to the Parquet schema for any operation with populated `details` and `details_json`.

## Target Code

- `internal/transform/operation.go:TransformOperation:85-98` — assigns both `OperationDetails` and `OperationDetailsJSON`
- `internal/transform/schema.go:OperationOutput:136-150` — JSON schema exposes both fields
- `internal/transform/schema_parquet.go:OperationOutputParquet:116-129` — Parquet schema only keeps `details`
- `internal/transform/parquet_converter.go:OperationOutput.ToParquet:147-160` — converter only serializes `OperationDetails`

## Evidence

The JSON schema really does export two names, and the Parquet schema retains only one. That made it plausible that `details_json` was being lost on the Parquet path.

## Anti-Evidence

`TransformOperation()` assigns `OperationDetailsJSON: outputDetails` from the exact same map used for `OperationDetails: outputDetails`, so the two JSON fields are aliases of one another rather than distinct payloads.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`details_json` is not distinct data. `TransformOperation()` writes the same `outputDetails` map into both `OperationDetails` and `OperationDetailsJSON`, and the Parquet path still preserves that payload once via `details`.

### Lesson Learned

Before treating a missing Parquet field as data loss, verify that the omitted JSON field is not just a second name for the same underlying value. Alias columns can look like omissions in a mechanical schema diff even when no information is actually lost.
