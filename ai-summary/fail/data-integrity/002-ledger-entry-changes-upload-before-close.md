# H004: Ledger Entry Change JSON Upload Races an Open File Handle

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each JSON batch file emitted by `export_ledger_entry_changes` should be fully flushed and closed before it is uploaded and deleted. The GCS object should contain the complete set of rows that were written for that batch.

## Mechanism

`exportTransformedData()` opens a batch output file, writes rows into it, and then calls `MaybeUpload(path)` without ever closing the handle. `GCS.UploadTo()` re-opens the pathname and deletes it after upload, so the command can upload a truncated snapshot of the still-open file while later buffered writes continue only to the original file descriptor, leaving GCS with an incomplete batch and no local file to retry from.

## Trigger

1. Run `export_ledger_entry_changes` with cloud upload enabled.
2. Export a batch large enough that buffered writes are still pending when `MaybeUpload()` re-opens the JSON file.
3. Compare the uploaded object with the eventual bytes written by the still-open descriptor; the uploaded GCS copy can be shorter than the final local stream.

## Target Code

- `cmd/export_ledger_entry_changes.go:309-318` — opens the JSON file and writes entries into it
- `cmd/export_ledger_entry_changes.go:368-372` — uploads without a prior `Close()`
- `cmd/command_utils.go:123-146` — `MaybeUpload()` re-opens the file path for upload
- `cmd/upload_to_gcs.go:32-35` — uploader reads the file by path
- `cmd/upload_to_gcs.go:69-72` — uploaded file is deleted immediately after upload

## Evidence

Every single-record export command closes `outFile` before uploading, but `export_ledger_entry_changes` is the lone outlier. The uploader is path-based, not descriptor-based, so it does not share the writer's open handle; it reads whatever bytes the filesystem has flushed to the pathname at upload time and then removes that pathname.

## Anti-Evidence

On some filesystems and for small batches, the open writer may have already flushed enough bytes that the uploaded file appears correct. The defect is timing-dependent, but it is still a real integrity risk because the code never establishes the required close-before-upload ordering.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not a direct duplicate, but root cause overlaps with `success/data-integrity/014-ledger-entry-changes-leaks-file-descriptors.md.gh-published`
**Failed At**: reviewer

### Trace Summary

Traced the full execution path from `exportTransformedData()` (line 295) through the write loop (lines 315-365) to `MaybeUpload()` (line 368). Confirmed the missing `Close()` is real. However, the hypothesis's core claim — that the upload can read truncated data due to "buffered writes still pending" — is incorrect. Go's `os.File.Write()` is a direct `write(2)` syscall wrapper with no application-level buffering. After each `Write()` returns, data is in the kernel page cache and fully visible to any reader opening the same path.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:295-377` — `exportTransformedData()`: the for loop writes all entries via `ExportEntry()` synchronously, then calls `MaybeUpload()` on line 368. All writes complete before the upload starts. No concurrent execution.
- `cmd/command_utils.go:55-87` — `ExportEntry()`: calls `outFile.Write(marshalled)` (line 78) and `outFile.WriteString("\n")` (line 82). Both are direct `os.File` method calls that invoke `write(2)` syscall with no buffering layer.
- `cmd/command_utils.go:31-52` — `MustOutFile()`: opens file with `os.OpenFile()`, returns raw `*os.File`. No `bufio.Writer` wrapping.
- `cmd/upload_to_gcs.go:25-74` — `UploadTo()`: opens the same path via `os.Open(path)` (line 32), reads it via `io.Copy()` (line 53). Since all writes completed before this call, the full file content is available.
- `cmd/command_utils.go:123-146` — `MaybeUpload()`: synchronous call, no goroutines.

### Why It Failed

The hypothesis's mechanism is wrong. It claims "buffered writes are still pending when `MaybeUpload()` re-opens the JSON file," but:

1. **No application-level buffering exists.** Go's `os.File.Write()` calls `poll.FD.Write()` → `syscall.Write()` directly. There is no `bufio.Writer` or other buffering wrapper anywhere in this code path. The data goes straight to the kernel.

2. **Execution is entirely sequential.** The write loop (lines 315-365) runs all iterations to completion, then line 368 calls `MaybeUpload()`. There are no goroutines, no async operations. Every byte written by `ExportEntry()` is committed to the kernel page cache before the upload begins.

3. **POSIX guarantees visibility.** After `write(2)` returns successfully, the written bytes are visible to any process (or any new file descriptor) that opens the same path. `Close()` is not required for data visibility — it only releases the file descriptor and ensures metadata finalization.

4. **The real impact of missing Close() is already captured.** The actual consequences of the missing `Close()` — file descriptor leaks and lost close-time I/O errors — are documented in `success/data-integrity/014-ledger-entry-changes-leaks-file-descriptors.md.gh-published`. That finding correctly identifies the FD leak mechanism rather than a non-existent truncation race.

### Lesson Learned

Go's `os.File.Write()` is an unbuffered syscall. Do not confuse it with buffered I/O (e.g., `bufio.Writer`). When evaluating missing `Close()` bugs, the real consequences are resource leaks and lost close-time errors, not data truncation — POSIX `write(2)` guarantees data visibility to other readers immediately upon return.
