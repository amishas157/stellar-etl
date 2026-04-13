# H003: Config-setting Parquet export flattens structured cost-parameter arrays into opaque strings

**Date**: 2026-04-13
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-config-settings --write-parquet`
exports `contract_cost_params_cpu_insns` and `contract_cost_params_mem_bytes`,
the Parquet row should preserve the same structured parameter data that the JSON
row carries: one element per cost-parameter entry with its `ExtV`, `ConstTerm`,
and `LinearTerm` values still accessible as fields rather than as an opaque blob.

## Mechanism

The JSON schema models both columns as `[]map[string]string`, but the Parquet
schema changes them to plain `string` fields and `ToParquet()` serializes the
arrays with `toJSONString(...)`. That means the Parquet export no longer carries
structured cost-parameter values at all; it carries a JSON-encoded text blob,
which silently changes the column type and breaks field-level analytics on the
same logical data.

## Trigger

1. Export any config-setting change row where either cost-parameter array is
   populated, such as `CONFIG_SETTING_CONTRACT_COST_PARAMS_CPU_INSTRUCTIONS`
   or `...MEM_BYTES`.
2. Compare the JSON output row against the Parquet row for the same ledger entry.
3. JSON exposes an array of parameter objects, while Parquet stores a single
   string like `[{"ExtV":"0","ConstTerm":"...","LinearTerm":"..."}]`.

## Target Code

- `internal/transform/schema.go:603-604` — JSON output defines both cost-param
  columns as structured `[]map[string]string`
- `internal/transform/schema_parquet.go:345-346` — Parquet schema narrows both
  columns to `string`
- `internal/transform/parquet_converter.go:386-387` — `ToParquet()` converts the
  structured arrays into JSON text with `toJSONString(...)`

## Evidence

This is not a generic "JSON and Parquet differ" complaint: the same config
setting row already has a stable, finite structure in JSON, and the converter
explicitly discards that structure by stringifying it. Unlike dynamic surfaces
such as `history_operations.details`, these two fields have a fixed, named shape
in the schema itself.

## Anti-Evidence

The flattening may have been an intentional workaround to avoid nested Parquet
records. But there is no code comment documenting that trade-off here, and the
schema comment still says the output aligns with the BigQuery config-settings
table, which makes the stringification look like a structural mismatch rather
than a clearly intentional alternate contract.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the Parquet conversion for `ContractCostParamsCpuInsns` and `ContractCostParamsMemBytes` from the JSON schema (`[]map[string]string`) through `parquet_converter.go`'s `toJSONString()` call to the Parquet schema (`string`). Then compared this against every other complex/nested field in the codebase to determine whether this is an anomaly or a consistent design pattern.

### Code Paths Examined

- `internal/transform/schema.go:603-604` — JSON schema defines cost params as `[]map[string]string`
- `internal/transform/schema_parquet.go:345-346` — Parquet schema defines them as `string` (BYTE_ARRAY/UTF8)
- `internal/transform/parquet_converter.go:386-387` — Converter calls `toJSONString()` on both fields
- `internal/transform/parquet_converter.go:153` — `OperationDetails` (`map[string]interface{}`) also stringified via `toJSONString()`
- `internal/transform/parquet_converter.go:282` — Effect `Details` (`map[string]interface{}`) also stringified via `toJSONString()`
- `internal/transform/schema_parquet.go:122,251` — Both `OperationDetails` and effect `Details` are `string` in Parquet, confirming the pattern

### Why It Failed

This is working-as-designed behavior, not a data correctness bug. The codebase has a **consistent design pattern**: every complex/nested type (`map[string]interface{}`, `[]map[string]string`) is serialized to a JSON string when converting to Parquet. This applies uniformly to:

1. `OperationOutput.OperationDetails` — `map[string]interface{}` → `string`
2. `EffectOutput.Details` — `map[string]interface{}` → `string`
3. `ConfigSettingOutput.ContractCostParamsCpuInsns` — `[]map[string]string` → `string`
4. `ConfigSettingOutput.ContractCostParamsMemBytes` — `[]map[string]string` → `string`

Meanwhile, simple flat arrays like `BucketListSizeWindow` (`[]uint64` → `[]int64` with `repetitiontype=REPEATED`) DO preserve their structure because `parquet-go` supports flat repeated primitives natively. The distinction is between flat arrays of primitives (preserved) and nested structures containing maps (stringified) — a library limitation, not a bug.

No data is lost or corrupted. The full structured content is preserved inside the JSON string. The hypothesis's own anti-evidence correctly identifies the root cause.

### Lesson Learned

JSON-to-string flattening for complex types in Parquet is a deliberate, codebase-wide design pattern driven by `parquet-go` limitations with nested map/struct types. All four usages of `toJSONString()` follow this pattern consistently. Future hypotheses about Parquet stringification should check whether the pattern is applied uniformly before flagging it as a bug.
