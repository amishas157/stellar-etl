# H003: Restored-key ledger-entry exports have no viable parquet path

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes` runs with `--export-restored-keys` and `--write-parquet`, the command should emit a parquet file whose restored-key rows match the JSON `restored_key` export for the same batch. The batch should not stop after producing only one of the two formats.

## Mechanism

The restored-key transform exists for JSON output, but `exportTransformedData` has no parquet switch case for `transform.RestoredKeyOutput`, and `internal/transform/schema_parquet.go` defines no `RestoredKeyOutputParquet`. That leaves `transformedResource` empty and `parquetSchema` nil for the `restored_key` resource, so the function reaches `WriteParquet(...)` without a schema capable of serializing the current batch's restored-key rows.

## Trigger

1. Run `export_ledger_entry_changes` with `--export-restored-keys` and `--write-parquet`.
2. Feed it any ledger range containing `LedgerEntryRestored` changes.
3. The JSON `restored_key` file is populated, but the parquet path for that same resource cannot be written correctly and the batch can terminate after partial output.

## Target Code

- `cmd/export_ledger_entry_changes.go:112-126` — appends `transform.RestoredKeyOutput` values into `transformedOutputs["restored_key"]`
- `cmd/export_ledger_entry_changes.go:321-364` — parquet type switch omits `transform.RestoredKeyOutput`
- `cmd/export_ledger_entry_changes.go:370-372` — still calls `WriteParquet(transformedResource, parquetPath, parquetSchema)`
- `internal/transform/schema.go:679-686` — JSON-side `RestoredKeyOutput` exists
- `internal/transform/schema_parquet.go:300-399` — no `RestoredKeyOutputParquet` exists alongside adjacent parquet schemas

## Evidence

The CLI explicitly exposes `export-restored-keys` and routes restored entries through `TransformRestoredKey`, so the JSON half of the feature is implemented. The parquet half has no matching schema type and no switch arm, unlike every other supported ledger-entry-change resource except the explicitly skipped claimable-balance case.

## Anti-Evidence

This issue is limited to parquet mode. JSON output for restored keys appears intentionally wired and should still contain correct rows when `--write-parquet` is not requested.
