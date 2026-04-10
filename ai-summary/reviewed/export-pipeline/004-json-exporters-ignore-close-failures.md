# H004: JSON exporters report success after `Close()` writeback failures

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Data loss or silent partial uploads
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If finalizing the JSON output file fails, the export command should surface that error and stop before printing success stats or uploading the path. A completed export should mean the JSONL file was fully committed to storage.

## Mechanism

The one-shot export commands call `outFile.Close()` and ignore its returned error, then immediately continue to `MaybeUpload(...)` or exit successfully. On filesystems that surface delayed allocation or writeback failures at close time, the command can treat a truncated or partially committed file as successful output and optionally upload it to cloud storage anyway.

## Trigger

Run `export_ledgers`, `export_transactions`, `export_contract_events`, or `export_token_transfer` on a filesystem that returns an error from `Close()` after the row writes have already returned successfully, especially with cloud upload enabled.

## Target Code

- `cmd/export_ledgers.go:63-68` — ignores `outFile.Close()` result and proceeds to upload
- `cmd/export_transactions.go:56-61` — same pattern
- `cmd/export_contract_events.go:57-65` — same pattern
- `cmd/export_token_transfers.go:57-62` — same pattern

## Evidence

Each command calls `outFile.Close()` as a bare statement rather than `if err := outFile.Close(); err != nil { ... }`. The export flow then prints stats and invokes `MaybeUpload(...)`, so the close result is the only missing gate between a failed final flush and a success-shaped completion path.

## Anti-Evidence

On healthy local disks, `Close()` usually succeeds, so the bug hides in routine runs. It needs a close-time filesystem error to manifest, but when it does, the current command paths have no way to distinguish a complete file from a failed final commit.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced `outFile.Close()` in all 9 one-shot export commands. Every one calls `outFile.Close()` as a bare statement, discarding the returned error. Execution then unconditionally proceeds to print success stats via `PrintTransformStats()` and upload via `MaybeUpload()`. This is a distinct error-swallowing path from reviewed H005 (which covers `ExportEntry` swallowing `Write()` errors during the row-writing loop). H004 covers the file-finalization stage, where `close(2)` can surface deferred writeback errors on network or certain local filesystems.

### Code Paths Examined

- `cmd/export_ledgers.go:63` — `outFile.Close()` bare statement, then line 66 prints stats, line 68 calls `MaybeUpload()`
- `cmd/export_transactions.go:56` — identical pattern: bare `outFile.Close()`, then stats on line 59, upload on line 61
- `cmd/export_contract_events.go:57` — bare `outFile.Close()`, stats on line 59, upload on line 61
- `cmd/export_token_transfers.go:57` — bare `outFile.Close()`, stats on line 60, upload on line 62
- `cmd/export_operations.go:56` — bare `outFile.Close()`, same unchecked pattern
- `cmd/export_effects.go:59` — bare `outFile.Close()`, same unchecked pattern
- `cmd/export_trades.go:61` — bare `outFile.Close()`, same unchecked pattern
- `cmd/export_assets.go:72` — bare `outFile.Close()`, same unchecked pattern
- `cmd/export_ledger_transaction.go:51` — bare `outFile.Close()`, same unchecked pattern
- `cmd/command_utils.go:123-146` — `MaybeUpload()` reads the file path and uploads to GCS, called after the unchecked Close()

### Findings

The bug is confirmed across **all 9 one-shot export commands**, broader than the 4 cited in the hypothesis. Every command follows the same pattern:

```go
outFile.Close()           // error discarded
cmdLogger.Info(...)       // success stats printed
PrintTransformStats(...)  // success summary
MaybeUpload(...)          // file uploaded to GCS
```

**Why this is a real bug despite Go's unbuffered I/O:**

1. **Go's `os.File.Write()` is unbuffered** — it invokes `write(2)` directly. So on standard local filesystems, data is in the kernel page cache by the time `Close()` is called. This means the practical trigger conditions are narrow.

2. **However, `close(2)` CAN surface real errors** in these scenarios:
   - **NFS/network filesystems**: NFS clients may buffer writes client-side. `close(2)` forces a flush to the server, which can return `EIO` or `ENOSPC`.
   - **Linux ext4 with writeback errors**: The kernel can report deferred I/O errors via `close(2)` that weren't visible at `write(2)` time.
   - **Any filesystem where the `write(2)` return doesn't guarantee persistence**: e.g., FUSE-based filesystems.

3. **Impact when triggered**: The command prints success stats, optionally uploads the file to GCS, and exits 0. No error is surfaced to monitoring, orchestration, or the operator. The local file may be incomplete on the persistent medium even though it appeared complete in the page cache at upload time.

4. **Complementary to H005**: The reviewed H005 finding covers `Write()` errors swallowed inside `ExportEntry()`. This finding covers `Close()` errors swallowed in the export commands. Even if H005 is fixed (making Write errors propagate), Close errors would still be silently ignored. Both are independent error-swallowing paths that need separate fixes.

5. **Scope wider than claimed**: The hypothesis cites 4 commands, but all 9 one-shot export commands share this defect. The 10th command (`export_ledger_entry_changes`) uses `exportTransformedData()` which has a separate issue — it never calls `Close()` at all (documented in fail/005).

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (create or append)
- **Setup**: Create a temporary file, write some data to it, then use `os.Pipe()` or a custom `*os.File` wrapper to simulate a `Close()` error. Alternatively, on Linux, open a file on an NFS mount or use a FUSE filesystem that errors on close. A simpler approach: use `os.NewFile()` with an already-closed file descriptor — calling `Close()` on it will return `EBADF`.
- **Steps**: In any one-shot export command (e.g., `export_ledgers`), after the write loop completes, the bare `outFile.Close()` call on line 63 should be changed to `if err := outFile.Close(); err != nil { cmdLogger.Fatal(...) }`. Verify that the current code path does NOT abort on `Close()` error and does proceed to `MaybeUpload()`.
- **Assertion**: Demonstrate that when `Close()` returns an error, the command still prints success stats and calls `MaybeUpload()`. After the fix, it should abort before reaching those calls.
