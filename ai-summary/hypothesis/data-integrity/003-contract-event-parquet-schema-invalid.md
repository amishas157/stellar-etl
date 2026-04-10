# H003: `ContractEventOutputParquet` cannot create a Parquet writer

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_contract_events --write-parquet` should serialize the same contract
event payload already present in JSON — including `topics`, `topics_decoded`,
`data`, and `data_decoded` — into a valid Parquet file. Enabling Parquet should
not stop the export before any rows are written.

## Mechanism

`ContractEventOutputParquet` declares `topics`/`topics_decoded` as UTF-8
`BYTE_ARRAY` columns but stores them as `[]interface{}`, and declares
`data`/`data_decoded` as scalar UTF-8 `BYTE_ARRAY` columns backed by
`interface{}`. `cmd.WriteParquet()` passes that malformed schema directly to
`writer.NewParquetWriter(...)`, and a probe against the production struct fails
immediately with `failed to create schema from tag map: type : not a valid Type
string`, so Parquet export aborts before any contract-event row can be written.

## Trigger

1. Run `export_contract_events --write-parquet` for any ledger range.
2. Reach the Parquet branch that calls `WriteParquet(..., new(transform.ContractEventOutputParquet))`.
3. Observe writer initialization fail before the first row is emitted, leaving
   no valid Parquet artifact for the requested contract-event export.

## Target Code

- `cmd/export_contract_events.go:63-65` — selects `ContractEventOutputParquet`
  for the Parquet export path
- `cmd/command_utils.go:162-180` — `WriteParquet()` calls
  `writer.NewParquetWriter(schema, ...)` and fatals on schema creation errors
- `internal/transform/schema_parquet.go:382-399` — declares
  `topics`/`topics_decoded`/`data`/`data_decoded` with incompatible
  `interface{}`-typed fields
- `internal/transform/parquet_converter.go:425-441` — copies raw interface
  payloads directly into the malformed Parquet row type
- `internal/transform/schema.go:640-657` — JSON schema shows the source fields
  really are dynamic interface payloads that need explicit Parquet encoding

## Evidence

An ad-hoc probe using the repository's production type
`new(transform.ContractEventOutputParquet)` and the same parquet-go writer path
returned `failed to create schema from tag map: type : not a valid Type
string` before any row write. The failure is consistent with the current schema:
there is no valid repeated/list annotation for `[]interface{}` fields, and the
scalar `interface{}` fields likewise do not tell parquet-go how to derive a
concrete physical type.

## Anti-Evidence

The JSON path is unaffected because `ExportEntry()` simply marshals the dynamic
payload with `json.Marshal`. If parquet-go had custom support for these exact
`interface{}` combinations the schema might be salvageable, but the current
writer initialization rejects it outright.
