# H001: Uploading JSON before `Close()` truncates the uploaded batch file

**Date**: 2026-04-10
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a batch JSON file is uploaded before the writer is closed, the uploaded object should still contain every row written so far. A missing `Close()` should not by itself make already-written bytes invisible to the uploader.

## Mechanism

I considered whether `export_ledger_entry_changes` might upload a partially written JSON file because it calls `MaybeUpload()` without first closing `outFile`. If true, GCS would receive a truncated but plausible file missing tail rows from the current batch.

## Trigger

Run `export_ledger_entry_changes` with GCS upload enabled and compare the uploaded JSON object with the local file contents after the batch finishes.

## Target Code

- `cmd/export_ledger_entry_changes.go:311-368` — writes then uploads without `outFile.Close()`
- `cmd/upload_to_gcs.go:32-55` — opens the path and streams it to GCS

## Evidence

The resource writer is never closed before upload, which is a real lifecycle smell and worth checking.

## Anti-Evidence

`UploadTo()` opens the path only after all `Write`/`WriteString` calls in the batch loop have already returned, and this code does not add any user-space buffering layer above `os.File`. That makes descriptor leakage plausible, but immediate truncation of already-written bytes is not well supported by the implementation.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The missing `Close()` is unlikely to hide bytes that were already written before `os.Open()` runs inside `UploadTo()`. The stronger issue on this path is descriptor leakage over time, not same-batch truncation.

### Lesson Learned

For local `os.File` I/O, separate "data visibility to a new reader" from "resource lifecycle correctness." Missing `Close()` can still be a bug, but not every missing close implies uploaded content is immediately stale or partial.
