# H002: Transaction Parquet export crashes on non-empty archived Soroban entry lists

**Date**: 2026-04-13
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Soroban transactions whose footprint extension contains archived entry
indexes, `export_transactions --write-parquet` should preserve those integers in
`soroban_resources_archived_entries` and continue writing the rest of the
transaction rows.

## Mechanism

`TransformTransaction()` builds `SorobanResourcesArchivedEntries` as
`[]uint32`, and `TransactionOutputParquet` keeps that field as `[]uint32` with a
repeated `INT32` tag. parquet-go accepts the schema, but when a row contains a
non-empty slice it tries to read those elements through the signed-int path and
fails with `reflect: call of reflect.Value.Int on uint32 Value`, aborting the
Parquet write on the first affected transaction.

## Trigger

1. Export any Soroban transaction whose `sorobanData.Ext.ResourceExt.ArchivedSorobanEntries`
   is non-empty.
2. Run `stellar-etl export_transactions --write-parquet ...`.
3. The Parquet writer initialization succeeds, but `writer.Write(record.ToParquet())`
   fails once it reaches that row.

## Target Code

- `internal/transform/transaction.go:171-174` — copies archived entry indexes
  into `outputSorobanArchivedEntries`
- `internal/transform/schema_parquet.go:59-66` — declares
  `SorobanResourcesArchivedEntries []uint32` as repeated `INT32`
- `internal/transform/parquet_converter.go:84-95` — forwards the `[]uint32`
  slice unchanged into the Parquet struct
- `cmd/command_utils.go:175-177` — `WriteParquet()` fatals on the first record
  write error

## Evidence

In-repo validation shows that a sample `TransactionOutputParquet` row with
non-empty `ExtraSigners` writes successfully, while the same row with only
`SorobanResourcesArchivedEntries = []uint32{1,2}` fails with
`reflect: call of reflect.Value.Int on uint32 Value`. That isolates the write
failure to the archived-entry slice rather than to the other repeated fields.

## Anti-Evidence

Transactions with an empty archived-entry list still Parquet-export correctly,
so this does not break every Soroban row. The hypothesis depends on legitimate
chain data reaching the archived-footprint path, but the source XDR field is
already wired through `TransformTransaction()`, so the trigger is real rather
than theoretical.
