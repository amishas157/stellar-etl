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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced all 7 export commands that support `--write-parquet` (ledgers, transactions, operations, trades, effects, assets, contract_events). Confirmed that 4 of 7 call `MaybeUpload(parquetPath)` before `WriteParquet()`, while the other 3 correctly call `WriteParquet()` first. Traced `MaybeUpload()` → `UploadTo()` which calls `os.Open(path)` to read the file, uploads it via GCS, then calls `deleteLocalFiles(path)` to remove the local copy. This means the upload reads whatever is at the path before `WriteParquet` creates the fresh file.

### Code Paths Examined

- `cmd/export_ledgers.go:70-72` — `MaybeUpload(parquetPath)` on line 71, then `WriteParquet()` on line 72. Reversed order.
- `cmd/export_transactions.go:63-65` — `MaybeUpload(parquetPath)` on line 64, then `WriteParquet()` on line 65. Reversed order.
- `cmd/export_operations.go:63-65` — `MaybeUpload(parquetPath)` on line 64, then `WriteParquet()` on line 65. Reversed order.
- `cmd/export_trades.go:68-70` — `MaybeUpload(parquetPath)` on line 69, then `WriteParquet()` on line 70. Reversed order.
- `cmd/export_effects.go:67-68` — `WriteParquet()` on line 67, then `MaybeUpload()` on line 68. **Correct order.**
- `cmd/export_assets.go:80-81` — `WriteParquet()` on line 80, then `MaybeUpload()` on line 81. **Correct order.**
- `cmd/export_contract_events.go:64-65` — `WriteParquet()` on line 64, then `MaybeUpload()` on line 65. **Correct order.**
- `cmd/command_utils.go:123-146` — `MaybeUpload()` delegates to `UploadTo()` when cloud provider is set.
- `cmd/upload_to_gcs.go:25-74` — `UploadTo()` opens the file at `path` (line 32), uploads to GCS (lines 47-58), then calls `deleteLocalFiles(path)` (line 71) removing the local file.

### Findings

Two distinct failure modes confirmed:

1. **First run (no stale file)**: `UploadTo()` calls `os.Open(parquetPath)` at line 32 of `upload_to_gcs.go`. The file does not exist because `WriteParquet()` hasn't run yet. `os.Open` returns an error, `UploadTo` returns it, and `MaybeUpload` calls `cmdLogger.Fatalf` at line 140 of `command_utils.go` — **process terminates without ever writing the parquet file**. The JSON output (already uploaded) succeeds but the parquet output is completely lost.

2. **Subsequent run (stale file at same path)**: `UploadTo()` successfully opens and uploads the **previous run's** parquet file to GCS. It then calls `deleteLocalFiles(parquetPath)` at line 71, removing the stale local copy. Control returns to the export command, which then calls `WriteParquet()` creating a fresh local file with current data. Result: **cloud has stale data from the previous run, local has fresh data from the current run** — silent data inconsistency.

The fix for all 4 commands is to swap the two lines within the `if commonArgs.WriteParquet` block so `WriteParquet()` executes before `MaybeUpload()`, matching the pattern used by `export_effects`, `export_assets`, and `export_contract_events`.

### PoC Guidance

- **Test file**: `cmd/export_ledgers_test.go` (or create a new integration test)
- **Setup**: Create a mock cloud storage implementation (or use the existing test infrastructure). Set up a test that enables both `--write-parquet` and cloud upload flags.
- **Steps**:
  1. Run `export_ledgers` with `--write-parquet` and cloud upload on a fresh parquet path. Observe fatal error because the file doesn't exist at upload time.
  2. Alternatively, write a sentinel parquet file to the output path before running the command. Run the export. Verify the uploaded file contains the sentinel (stale) data rather than the newly transformed data.
- **Assertion**: Assert that the content uploaded to cloud storage matches the content produced by `WriteParquet()` for the current ledger range, not stale data from a previous file at the same path. The simplest structural test: verify `WriteParquet()` is called before `MaybeUpload()` by checking that the parquet file exists and has non-zero size at the point `MaybeUpload` reads it.
