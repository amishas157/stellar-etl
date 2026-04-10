# H004: Four export commands upload stale or nonexistent Parquet files before writing them

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Data loss or stale Parquet uploads
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `--write-parquet` and cloud upload are both enabled, the exporter should write the fresh local Parquet file first and only then upload that finished file. Reusing an output path should never cause a previous run's Parquet contents to be uploaded for a new ledger range.

## Mechanism

`export_ledgers`, `export_transactions`, `export_operations`, and `export_trades` all call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. If `parquetPath` does not exist yet, upload fails before the new file is created; if a stale file from a previous run still exists at that path, the command uploads old data and then silently overwrites the local Parquet afterward, leaving cloud and local outputs inconsistent.

## Trigger

Run any of the affected commands with `--write-parquet` plus cloud upload enabled. The stale-data variant is easiest to reproduce by reusing a Parquet output path that already contains a prior export.

## Target Code

- `cmd/export_ledgers.go:68-72` — uploads `parquetPath` before `WriteParquet`
- `cmd/export_transactions.go:61-65` — uploads `parquetPath` before `WriteParquet`
- `cmd/export_operations.go:61-65` — uploads `parquetPath` before `WriteParquet`
- `cmd/export_trades.go:66-70` — uploads `parquetPath` before `WriteParquet`

## Evidence

Sibling commands such as `export_effects`, `export_assets`, and `export_contract_events` call `WriteParquet()` first and upload second, which shows the intended lifecycle. The four affected commands are the outliers in an otherwise consistent export template.

## Anti-Evidence

If cloud upload is disabled, this bug does not affect local files. When the target Parquet path is brand new, the first failure mode is loud, but path reuse converts the bug into silent stale-data corruption.
