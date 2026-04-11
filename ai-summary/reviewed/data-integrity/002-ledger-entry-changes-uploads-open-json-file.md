# H002: `export_ledger_entry_changes` uploads JSON batch files before closing them

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each JSON batch file written by `export_ledger_entry_changes` should be closed successfully before any cloud upload begins, so the uploaded object reflects the full on-disk contents and close-time I/O errors are surfaced. The command should not leave per-resource output descriptors open across uploads.

## Mechanism

`exportTransformedData()` opens a fresh `*os.File` for every resource, writes JSON rows through `ExportEntry()`, and then immediately calls `MaybeUpload(path)` without any `outFile.Close()` or close-error check. That makes the cloud upload race the final file flush/metadata update and can silently publish truncated or stale JSON objects while the local file descriptor remains open until process exit.

## Trigger

Run `export_ledger_entry_changes` with a cloud provider configured on any non-empty batch that emits at least one JSON resource file, especially on slower disks or remote filesystems where close/flush latency is observable.

## Target Code

- `cmd/export_ledger_entry_changes.go:309-318` — opens the per-resource JSON file and writes entries
- `cmd/export_ledger_entry_changes.go:368-372` — uploads the JSON path without closing the file first
- `cmd/command_utils.go:31-52` — `MustOutFile()` returns an open `*os.File`
- `cmd/command_utils.go:55-86` — `ExportEntry()` writes directly to that descriptor
- `cmd/command_utils.go:123-145` — `MaybeUpload()` immediately reads/uploads the named file

## Evidence

Sibling export commands such as `export_transactions`, `export_trades`, and `export_effects` all call `outFile.Close()` before `MaybeUpload()`, which makes `export_ledger_entry_changes` a clear lifecycle outlier. In this command the file is never closed inside `exportTransformedData()` at all, so every resource upload happens against a still-open descriptor.

## Anti-Evidence

Some local filesystems will make written bytes visible quickly enough that many uploads may still appear correct in practice, especially for small batches. If the runtime exits immediately after upload, the descriptor leak may not be user-visible locally even though the remote object was uploaded from an open file.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`exportTransformedData()` iterates over resource types, opening a fresh `*os.File` via `MustOutFile(path)` for each (line 311), writing JSON rows via `ExportEntry()` (line 316), then calling `MaybeUpload()` (line 368) without ever closing `outFile`. The `MaybeUpload` → `GCS.UploadTo()` path re-opens the same file path for reading (`os.Open(path)` at upload_to_gcs.go:32), copies its content to GCS, then deletes the local file (`deleteLocalFiles(path)` at upload_to_gcs.go:71) — all while the original write descriptor is still open. All 9 sibling export commands (`export_transactions`, `export_operations`, `export_ledgers`, `export_effects`, `export_trades`, `export_assets`, `export_contract_events`, `export_token_transfers`, `export_ledger_transaction`) call `outFile.Close()` before `MaybeUpload()`.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-377` — iterates resources, opens file at 311, writes at 316, uploads at 368; no `Close()` call anywhere in this function
- `cmd/command_utils.go:MustOutFile:31-53` — returns an open `*os.File` with O_RDWR|O_CREATE|O_TRUNC
- `cmd/command_utils.go:ExportEntry:55-87` — writes JSON + newline to the open descriptor via `outFile.Write()` and `outFile.WriteString()`
- `cmd/command_utils.go:MaybeUpload:123-146` — dispatches to `GCS.UploadTo()` which opens the file again by path
- `cmd/upload_to_gcs.go:UploadTo:25-74` — `os.Open(path)` at line 32, `io.Copy` at line 53, `deleteLocalFiles(path)` at line 71
- `cmd/export_transactions.go:56-61` — representative sibling: `outFile.Close()` at 56, `MaybeUpload()` at 61
- All 9 sibling commands confirmed to follow the Close-then-Upload pattern

### Findings

1. **File descriptor leak**: Each resource type per batch leaks one `*os.File` descriptor. With 10+ resource types enabled and continuous batch processing, this accumulates over the process lifetime. The descriptors are never reclaimed until process exit.

2. **Missing close-error detection**: `os.File.Close()` can return errors (e.g., deferred write failure on NFS, disk-full on final metadata flush). By never calling Close, these errors are silently suppressed.

3. **Upload reads unflushed file**: On local filesystems with a unified page cache (Linux, macOS), data written via `outFile.Write()` is typically visible to a separate `os.Open()` reader even before close. However, this is an OS implementation detail, not a POSIX guarantee. On network filesystems (NFS, FUSE-mounted storage), the data may not be visible to another open until the writing descriptor is closed and flushed.

4. **Delete of open file**: `deleteLocalFiles(path)` unlinks the file while `outFile` still holds an open descriptor, creating a zombie file descriptor pointing to deleted inode.

5. **Clear lifecycle outlier**: 9 of 10 export commands follow the `MustOutFile → Write → Close → MaybeUpload` pattern. `export_ledger_entry_changes` is the sole exception.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` (create if needed, or append to existing test infrastructure)
- **Setup**: Create a mock `exportTransformedData` call with a temporary directory, a small set of transformed outputs, and a mock or no-op cloud provider. Track whether `outFile.Close()` is called before the upload step.
- **Steps**: (1) Call `exportTransformedData` with at least one resource type containing entries. (2) Instrument or wrap `MustOutFile` to return a file whose `Close()` method records whether it was called. (3) Verify `Close()` is called before `MaybeUpload()` is invoked.
- **Assertion**: Demonstrate that `outFile.Close()` is never called in the current code path. Compare with the sibling command pattern by showing the Close call exists in `export_transactions.go` but is absent from `export_ledger_entry_changes.go`. A simple `grep -c 'outFile.Close' cmd/export_ledger_entry_changes.go` returning 0 versus non-zero for siblings is sufficient to confirm the structural defect.
