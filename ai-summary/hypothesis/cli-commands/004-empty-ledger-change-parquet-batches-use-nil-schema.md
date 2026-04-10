# H004: Empty ledger-entry-change batches still enter parquet generation with a nil schema

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a selected ledger-entry resource has zero rows in a batch, `export_ledger_entry_changes` should either skip parquet creation for that batch or create a valid empty parquet file using the resource's schema. An empty batch should not break the whole export.

## Mechanism

`transformedOutputs` is pre-populated with empty slices for every selected resource before the batch is examined. In `exportTransformedData`, the parquet schema is only assigned inside the `for _, o := range output` loop. When `output` is empty, `skip` stays false, `parquetSchema` stays nil, and the function still calls `WriteParquet(...)`; the parquet writer code dereferences its schema handler during finalization, so an empty-but-selected resource can stop the export instead of yielding an empty result set.

## Trigger

Run `export_ledger_entry_changes` with `--write-parquet=true` for a selected resource whose batch has no rows, such as `--export-config-settings=true` on ledgers `59834400-59834415` (the same empty-range case already covered by JSON-only tests). The JSON path creates an empty file, but the parquet path reaches `WriteParquet` without ever choosing a schema.

## Target Code

- `cmd/export_ledger_entry_changes.go:103-109` — selected resources are initialized even when the batch contains no matching changes
- `cmd/export_ledger_entry_changes.go:304-364` — schema selection happens only inside the per-row loop
- `cmd/export_ledger_entry_changes.go:370-372` — parquet write still runs with `skip == false`
- `cmd/command_utils.go:162-180` — `WriteParquet` does not guard against a nil schema before creating the writer

## Evidence

There is already a JSON-only regression test proving empty config-setting batches are expected and valid output (`cmd/export_ledger_entry_changes_test.go:138-144`). In the current implementation, the parquet schema is inferred from the first row instead of from the resource type, so empty output slices have no way to initialize `parquetSchema` before `WriteParquet` is invoked.

## Anti-Evidence

If at least one row of a supported type appears in the batch, the switch sets `parquetSchema` and the parquet path can proceed normally. This only manifests on empty batches for selected resource types.
