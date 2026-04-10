# H005: Several export commands upload Parquet paths before writing the Parquet file

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `--write-parquet` and cloud upload are both enabled, each export command should first call `WriteParquet(...)` to materialize the current batch on disk, then call `MaybeUpload(...)` to send that just-written file to cloud storage. Remote Parquet artifacts should always match the current export run.

## Mechanism

`export_transactions`, `export_operations`, `export_ledgers`, and `export_trades` call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. `UploadTo()` immediately opens the local path for reading, so a fresh run fails because the parquet file does not exist yet, while a rerun against an old local file uploads stale data from the previous run and then deletes it before the current run writes the new file. Sister commands like `export_effects`, `export_assets`, and `export_ledger_entry_changes` use the correct write-then-upload order, which makes the outlier behavior especially suspicious.

## Trigger

Run any affected command with Parquet output and GCS upload enabled, for example `export_transactions --write-parquet --cloud-provider gcp ...`. On a fresh path, upload fails before the parquet writer runs; on a reused path, the remote object receives stale parquet bytes from the prior run instead of the current export.

## Target Code

- `cmd/export_transactions.go:63-65` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/export_operations.go:63-65` — same reversed order
- `cmd/export_ledgers.go:70-72` — same reversed order
- `cmd/export_trades.go:68-70` — same reversed order
- `cmd/upload_to_gcs.go:25-35` — `UploadTo()` opens the local file immediately, so ordering directly controls correctness

## Evidence

The affected commands issue `MaybeUpload(..., parquetPath)` first and only then invoke `WriteParquet(...)`. In contrast, `export_effects.go`, `export_assets.go`, `export_contract_events.go`, and `export_ledger_entry_changes.go` write the parquet file before uploading it, which matches the expected lifecycle.

## Anti-Evidence

The buggy branch is only reached when both cloud upload and Parquet output are enabled. Local-only runs or JSON-only runs are unaffected.
