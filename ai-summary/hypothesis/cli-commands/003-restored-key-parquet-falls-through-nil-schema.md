# H003: Restored-key parquet export falls through the type switch and reaches WriteParquet with no schema

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes` is run with `--export-restored-keys --write-parquet`, the command should either write a valid restored-key parquet file or explicitly skip parquet for restored keys. It should not attempt parquet generation with an unset schema.

## Mechanism

The command adds `restored_key` rows to `transformedOutputs`, but `exportTransformedData` has no `case transform.RestoredKeyOutput` in its parquet type switch and there is no `RestoredKeyOutputParquet` schema. That leaves `parquetSchema` nil and `skip` false, so the function proceeds into `WriteParquet(transformedResource, parquetPath, parquetSchema)` with no supported schema for that resource, breaking parquet export for an otherwise valid restored-key batch.

## Trigger

Run `export_ledger_entry_changes` on a range containing restored ledger keys, with both `--export-restored-keys=true` and `--write-parquet=true`. JSON output is produced, then the parquet branch attempts to serialize `restored_key` rows without ever selecting a parquet schema for them.

## Target Code

- `cmd/export_ledger_entry_changes.go:112-125` — restored keys are transformed and queued for export
- `cmd/export_ledger_entry_changes.go:321-364` — parquet type switch omits `transform.RestoredKeyOutput`
- `cmd/export_ledger_entry_changes.go:370-372` — parquet write proceeds when `skip` is still false
- `internal/transform/schema.go:679-686` — restored-key JSON schema exists

## Evidence

The repository defines `RestoredKeyOutput` and enables `--export-restored-keys`, but searches across `internal/transform` show no `RestoredKeyOutputParquet` type and no `ToParquet` implementation. Every supported ledger-entry-changes parquet resource has an explicit type-switch arm; restored keys do not.

## Anti-Evidence

If `--write-parquet` is omitted, restored-key JSON export is unaffected. This is specifically a parquet-mode gap rather than a problem with restored-key transformation itself.
