# H004: Restored-key JSON rows are dropped into an empty parquet file

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_ledger_entry_changes --export-restored-keys --write-parquet` produces restored-key JSON rows for a batch, the corresponding parquet file should contain the same restored-key rows.

## Mechanism

The restored-key path appends `transform.RestoredKeyOutput` rows into `transformedOutputs["restored_key"]`, but the downstream Parquet type switch has no `RestoredKeyOutput` case. As a result, the JSON file is populated while `transformedResource` stays empty and `WriteParquet()` emits a zero-row parquet file, silently erasing every restored key from the Parquet side of the export.

## Trigger

Run `stellar-etl export_ledger_entry_changes --export-restored-keys true --write-parquet ...` over any batch that contains restored ledger entries, such as the repository's existing restored-key test range `58764192-58764193`.

## Target Code

- `cmd/export_ledger_entry_changes.go:97-100` — maps `--export-restored-keys` to the `restored_key` output bucket
- `cmd/export_ledger_entry_changes.go:112-125` — transforms restored ledger changes into `transform.RestoredKeyOutput` rows
- `cmd/export_ledger_entry_changes.go:321-364` — Parquet type switch omits `transform.RestoredKeyOutput`
- `cmd/export_ledger_entry_changes.go:370-372` — still calls `WriteParquet()` after the unsupported rows were never appended
- `internal/transform/restored_key.go:40-46` — defines the JSON row that currently has no Parquet handling
- `cmd/export_ledger_entry_changes_test.go:97-101` — demonstrates that restored-key JSON export is a live, covered path

## Evidence

The export mapping and transformation loop clearly populate restored-key JSON output today, but the Parquet switch never mentions `RestoredKeyOutput`. I also confirmed that the parquet library accepts `WriteParquet(nil, path, nil)` for zero rows, which turns this into silent data loss rather than an obvious crash.

## Anti-Evidence

This is Parquet-only: the JSON batch remains correct, and users who do not enable `--write-parquet` will not see the issue. The actual on-disk behavior depends on the parquet library tolerating the empty write, but that tolerance is exactly what makes the drop silent.
