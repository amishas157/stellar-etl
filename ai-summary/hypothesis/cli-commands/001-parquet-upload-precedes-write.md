# H001: Parquet upload happens before current batch is written

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `--write-parquet` and GCS upload are both enabled, each export command should serialize the current ledger range to the local parquet file first, then upload that exact file. The JSON and Parquet objects for a given run should represent the same ledgers and the same row set.

## Mechanism

`export_ledgers`, `export_transactions`, `export_operations`, and `export_trades` call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. On a first run, the upload path does not exist yet, so upload fails before any parquet file is created. On a repeated run that reuses the same local path, `UploadTo` opens the stale parquet from the previous export, uploads it, and deletes it before the new batch is written; the fresh parquet is then created locally but never uploaded.

## Trigger

1. Run `export_ledgers`, `export_transactions`, `export_operations`, or `export_trades` with `--write-parquet` and `--cloud-provider gcp`.
2. Use a parquet output path that either does not exist yet (first run) or still contains the previous run's parquet file.
3. Observe that GCS either receives no parquet object for the current batch or receives the previous batch's parquet while JSON reflects the current batch.

## Target Code

- `cmd/export_ledgers.go:Run:68-73` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/export_transactions.go:Run:61-66` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/export_operations.go:Run:61-66` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/export_trades.go:Run:66-70` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/upload_to_gcs.go:UploadTo:32-35,47-73` — opens the existing local file and deletes it after upload

## Evidence

The four commands above all use the reversed order `MaybeUpload(parquetPath)` then `WriteParquet(...)`. `UploadTo` reads whatever file is already present at `path`, verifies the object exists in GCS, and then removes the local file via `deleteLocalFiles(path)`.

## Anti-Evidence

Other exporters in the same package use the correct order. `export_assets`, `export_effects`, `export_contract_events`, and `export_ledger_entry_changes` all write the parquet file before uploading it, which suggests the intended behavior is well understood elsewhere in the subsystem.
