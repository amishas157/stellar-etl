# H003: `export_ledger_entry_changes` uploads JSON batches before closing the file

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Data loss or silent partial uploads
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each batch file produced by `export_ledger_entry_changes` should be fully flushed and closed before cloud upload starts. The uploaded object should match the completed local JSON file byte-for-byte.

## Mechanism

`exportTransformedData()` opens `outFile`, writes rows with `ExportEntry()`, and then immediately calls `MaybeUpload(..., path)` without ever calling `outFile.Close()`. On systems where buffered writes have not been flushed yet, GCS upload can observe a shorter or empty file while the command continues as if the batch exported successfully.

## Trigger

Run `export_ledger_entry_changes` with any export flag and cloud upload enabled so that at least one JSON batch file is written and uploaded.

## Target Code

- `cmd/export_ledger_entry_changes.go:309-318` — opens the batch output file and writes rows
- `cmd/export_ledger_entry_changes.go:368-372` — uploads `path` without closing the JSON file handle first
- `cmd/command_utils.go:123-146` — `MaybeUpload()` immediately reads the filesystem path it is given

## Evidence

Sibling one-shot export commands explicitly call `outFile.Close()` before uploading their JSON path, but `exportTransformedData()` has no close on the JSON handle at all. The parquet branch writes before upload and therefore does not share the same exact ordering problem.

## Anti-Evidence

Small batches may appear correct when the OS flushes eagerly, so the bug can hide in light testing. The issue only manifests when cloud upload is enabled, but that is the production path this helper is designed to support.
