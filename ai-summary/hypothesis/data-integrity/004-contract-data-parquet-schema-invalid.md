# H004: `ContractDataOutputParquet` cannot create a Parquet writer

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-contract-data --write-parquet`
encounters contract-data rows, it should write a valid Parquet file preserving
the same `key`, `key_decoded`, `val`, and `val_decoded` payloads that the JSON
export already emits for those ledger entries.

## Mechanism

`ContractDataOutputParquet` models `key`, `key_decoded`, `val`, and
`val_decoded` as plain `interface{}` fields tagged with `type=MAP,
convertedtype=MAP,...`. `cmd.WriteParquet()` feeds that struct to
`writer.NewParquetWriter(...)`, and a probe against the production schema fails
immediately with `failed to create schema from tag map: type MAP: not a valid
Type string`, so any contract-data Parquet export aborts before the first row
is written.

## Trigger

1. Run `export_ledger_entry_changes --export-contract-data --write-parquet` on
   any batch that contains contract-data ledger entries.
2. Reach `exportTransformedData()`'s `transform.ContractDataOutput` branch.
3. Observe Parquet writer initialization fail for
   `new(transform.ContractDataOutputParquet)`, leaving JSON output only and no
   usable Parquet artifact for the requested contract-data batch.

## Target Code

- `cmd/export_ledger_entry_changes.go:321-347` — selects
  `ContractDataOutputParquet` for contract-data batches
- `cmd/export_ledger_entry_changes.go:370-372` — calls `WriteParquet()` for the
  selected schema
- `cmd/command_utils.go:162-180` — fatals when parquet-go rejects schema
  creation
- `internal/transform/schema_parquet.go:260-281` — declares MAP-typed
  `interface{}` fields for `key`/`val` payloads
- `internal/transform/parquet_converter.go:292-314` — copies the dynamic JSON
  payloads directly into those Parquet fields

## Evidence

An ad-hoc probe using `new(transform.ContractDataOutputParquet)` and the
production parquet-go writer returned `failed to create schema from tag map:
type MAP: not a valid Type string` before any row data was written. The schema
tags ask parquet-go to derive a MAP from opaque `interface{}` fields without a
concrete repeated key/value layout, which is exactly the kind of tag/field
combination the writer rejects.

## Anti-Evidence

JSON export of contract data still works because the dynamic fields are simply
marshaled as JSON. The existing Parquet omission of `ledger_key_hash_base_64`
is a separate bug pattern; even after that omission is addressed, the current
schema definition still prevents any contract-data Parquet writer from being
constructed.
