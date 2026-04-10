# H001: Parquet upload runs before local parquet generation in four export commands

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `--write-parquet` and cloud upload are both enabled, each command should first generate the parquet file for the current ledger range and then upload that freshly written file. The object stored in GCS should match the JSON export from the same run, not a prior run's parquet bytes.

## Mechanism

`export_ledgers`, `export_transactions`, `export_operations`, and `export_trades` all call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. `UploadTo` opens whatever file already exists at `parquetPath`, uploads it to GCS, and deletes it locally; only after that does the command create the new parquet file. On a rerun that reuses the same parquet path, GCS silently receives stale parquet content from the previous export while the current run leaves the correct parquet only on local disk.

## Trigger

1. Run one of the affected commands with `--write-parquet`, `--cloud-provider gcp`, and a fixed `--parquet-output` path.
2. Leave the old parquet file in place from an earlier export for a different ledger range or different network.
3. Re-run the command for a new ledger range using the same parquet path.
4. The uploaded GCS object contains the previous run's rows, while the newly generated local parquet file contains the current run's rows.

## Target Code

- `cmd/export_ledgers.go:68-72` — uploads `parquetPath` before generating parquet
- `cmd/export_transactions.go:61-65` — same ordering bug on transaction parquet export
- `cmd/export_operations.go:61-65` — same ordering bug on operation parquet export
- `cmd/export_trades.go:66-70` — same ordering bug on trade parquet export
- `cmd/upload_to_gcs.go:25-35` — upload reads the existing local path as-is
- `cmd/upload_to_gcs.go:71-71` — upload deletes the local file after sending it

## Evidence

The unaffected sibling commands (`export_assets`, `export_effects`, `export_contract_events`, and `export_ledger_entry_changes`) all call `WriteParquet(...)` before `MaybeUpload(...)`, establishing the intended template. The buggy four commands reverse that order, and `UploadTo` unconditionally opens the current file at `path`, so a leftover parquet file is enough to make the cloud copy stale but plausible.

## Anti-Evidence

If the parquet path does not already exist, the command fails loudly because `UploadTo` cannot open the file. The silent-corruption path requires a rerun or reused workspace where an older parquet file already exists at the same path.
