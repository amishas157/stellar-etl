# H004: Empty ledger-entry-change resource batches reach Parquet writing with no schema

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --write-parquet` selects a resource type that happens to have zero rows in a given batch, the command should either write an empty Parquet file with that resource's schema or explicitly skip Parquet for that empty batch. It should not attempt a schema-less Parquet write after already producing the JSON side of the batch.

## Mechanism

The command pre-populates `transformedOutputs` with empty slices for every selected resource before it knows whether the batch contains any rows. In `exportTransformedData()`, the Parquet schema is inferred only inside the per-row type switch, so an empty output slice leaves `parquetSchema == nil` and `skip == false`. `WriteParquet()` then calls `writer.NewParquetWriter(..., nil, 1)`, which leaves `SchemaHandler` unset; `WriteStop()` later dereferences that nil schema handler, aborting the run after the JSON file for that resource was already written (and potentially uploaded).

## Trigger

Run `export_ledger_entry_changes --write-parquet` on any multi-resource range where at least one selected resource has zero rows in a batch, such as a batch with account updates but no signer deltas. The command should emit an empty-but-valid parquet artifact or skip it cleanly, but this path should instead fail once it reaches that empty resource's Parquet write.

## Target Code

- `cmd/export_ledger_entry_changes.go:90-109` - selected resource keys are seeded into `transformedOutputs` as empty slices before transformation
- `cmd/export_ledger_entry_changes.go:304-372` - `exportTransformedData()` infers `parquetSchema` only from actual rows, then writes Parquet whenever `skip` is still false
- `cmd/command_utils.go:162-180` - `WriteParquet()` forwards the possibly nil schema into parquet-go and always calls `WriteStop()`
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/writer/writer.go:83-100` - `NewParquetWriter()` leaves `SchemaHandler` unset when `obj == nil`
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/writer/writer.go:129-138` - `WriteStop()` immediately calls `RenameSchema()`, which dereferences `pw.SchemaHandler`

## Evidence

The resource map is seeded before any transformations, so empty per-resource batches are part of normal control flow. But the schema-selection switch only runs inside `for _, o := range output`, leaving no schema at all for empty outputs even though the code still unconditionally enters the Parquet branch.

## Anti-Evidence

This does not fire if `--write-parquet` is disabled or if every selected resource gets at least one row in every batch. It is also distinct from the already-known restored-key and claimable-balance Parquet gaps, because it affects otherwise supported resource types when their current batch is empty.
