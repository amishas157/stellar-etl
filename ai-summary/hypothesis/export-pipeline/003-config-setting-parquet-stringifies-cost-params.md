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
