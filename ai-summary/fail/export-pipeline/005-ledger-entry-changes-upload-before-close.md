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

---

## Review

**Verdict**: NEEDS_REFINEMENT
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Failed At**: reviewer

### What's Wrong

The hypothesis correctly identifies that `exportTransformedData()` never calls `outFile.Close()`, which is inconsistent with all 9 sibling export commands. However, the claimed data-loss mechanism is wrong.

In Go, `*os.File.Write()` is **not buffered in userspace** — it directly invokes the `write(2)` syscall. Data written via `outFile.Write()` is immediately visible in the kernel page cache. When `UploadTo()` subsequently calls `os.Open(path)` and `io.Copy()` on line 32-53 of `upload_to_gcs.go`, it will always see the complete file contents, because POSIX guarantees read coherence with prior `write(2)` calls on the same file.

Therefore, the specific claim — "GCS upload can observe a shorter or empty file" — is incorrect. The uploaded file will always contain complete data.

### Alternative Angle

The missing `Close()` IS a real bug, but its impact is **file descriptor leaks**, not data loss:

1. **File descriptor leak**: `exportTransformedData()` iterates over `transformedOutput` (up to 10 entity types per batch). Each iteration opens a new `*os.File` via `MustOutFile(path)` at line 311 but never closes it. Over sustained operation with many batches, this accumulates leaked file descriptors.

2. **Interaction with `deleteLocalFiles()`**: `UploadTo()` calls `deleteLocalFiles(path)` at line 71 of `upload_to_gcs.go`, which removes the file from the filesystem. But the unclosed `outFile` still holds the file descriptor, keeping the inode and disk space allocated until the GC finalizer runs or the process exits.

3. **Consistency defect**: All 9 sibling export commands (`export_ledgers.go:63`, `export_transactions.go:56`, `export_operations.go:56`, `export_effects.go:59`, `export_trades.go:61`, `export_assets.go:72`, `export_contract_events.go:57`, `export_token_transfers.go:57`, `export_ledger_transaction.go:51`) call `outFile.Close()` before `MaybeUpload()`. `exportTransformedData()` is the sole outlier.

A refined hypothesis should target the file descriptor leak and resource accumulation under sustained batch operation, with severity likely Medium.

### Additional Code Paths

- `cmd/export_ledger_entry_changes.go:295-377` — full `exportTransformedData()` function, verify no deferred close exists
- `cmd/upload_to_gcs.go:71` — `deleteLocalFiles(path)` called while `outFile` is still open
- All sibling export commands for the consistent `Close()` → `MaybeUpload()` pattern
