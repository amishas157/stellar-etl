# H003: `export_ledger_entry_changes` can lose later batches after leaking file descriptors

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each per-resource batch file opened by `export_ledger_entry_changes` should be closed once the batch has been written and optionally uploaded. Long-running exports over many batches should therefore be able to emit every expected file instead of failing partway through with open-file exhaustion.

## Mechanism

`MustOutFile()` first calls `createOutputFile()`, which uses `os.Create()` without closing the returned descriptor, then opens the same path again with `os.OpenFile()`. `exportTransformedData()` never closes that second `outFile` at all, so every resource file in every batch leaks at least one live descriptor and usually two. On sufficiently long exports, later file opens fail with `EMFILE`, which aborts remaining resources/batches and leaves a partial export on disk or in cloud storage.

## Trigger

Run `export_ledger_entry_changes` over a long range with a small `--batch-size` and multiple `--export-*` resource types enabled. As the command iterates batches, it opens a new JSON file for each resource and never closes it; once the process crosses the OS open-file limit, later `MustOutFile()` or parquet opens will fail and the remaining batch artifacts will be missing.

## Target Code

- `cmd/command_utils.go:19-25` — `createOutputFile()` calls `os.Create()` and drops the returned file handle
- `cmd/command_utils.go:31-52` — `MustOutFile()` opens the same path again and returns a second handle
- `cmd/export_ledger_entry_changes.go:309-372` — `exportTransformedData()` writes, uploads, and returns without ever calling `outFile.Close()`

## Evidence

The leak is visible directly in the control flow: after `outFile := MustOutFile(path)`, the function writes rows, optionally uploads JSON and Parquet, and exits the loop body without any close call. This command is uniquely susceptible because it opens one file per resource per batch, so descriptor growth scales with both range length and number of exported entity types rather than staying at a single file per process.

## Anti-Evidence

Short runs may never accumulate enough open files to hit the OS limit, and process exit will eventually release the descriptors. But that only masks the issue on small workloads; the bug still turns perfectly valid large exports into partial datasets once the descriptor ceiling is reached.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — prior fail/005 investigated a different hypothesis (upload-before-close → partial GCS uploads) and was rejected for the wrong mechanism; its lesson learned explicitly called out FD leaks as the real issue and suggested refiling. This hypothesis correctly targets the FD leak itself.

### Trace Summary

`exportTransformedData()` iterates over each resource type in `transformedOutput`, calling `MustOutFile(path)` on line 311 which returns an `*os.File`. The function uses this handle for `ExportEntry()` writes, then calls `MaybeUpload()` and optionally `WriteParquet()`, but never calls `outFile.Close()`. Meanwhile, `MustOutFile()` internally calls `createOutputFile()` which leaks an additional FD via `os.Create()` with a discarded return value. Every other export command in the `cmd/` package calls `outFile.Close()` after writing — this command is the sole outlier. Additionally, `UploadTo()` opens a third FD via `os.Open(path)` for reading during upload and also never closes it.

### Code Paths Examined

- `cmd/command_utils.go:createOutputFile:19-29` — `os.Create(filepath)` return assigned to `_`, leaking one FD per new file
- `cmd/command_utils.go:MustOutFile:31-52` — calls `createOutputFile` then `os.OpenFile`, returning second handle; caller must close
- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-377` — loops over resource types, opens file at line 311, writes entries, uploads, writes parquet, but NO `outFile.Close()` anywhere in the loop body or after it
- `cmd/upload_to_gcs.go:UploadTo:25-74` — opens a third FD via `os.Open(path)` at line 32 without closing `reader`; calls `deleteLocalFiles(path)` which unlinks the file but does not release open FDs
- `cmd/export_ledgers.go:63`, `cmd/export_transactions.go:56`, `cmd/export_operations.go:56`, `cmd/export_effects.go:59`, `cmd/export_trades.go:61`, `cmd/export_assets.go:72`, `cmd/export_contract_events.go:57`, `cmd/export_token_transfers.go:57`, `cmd/export_ledger_transaction.go:51` — all nine one-shot export commands call `outFile.Close()`, confirming `export_ledger_entry_changes` is the sole outlier

### Findings

1. **Missing `outFile.Close()` in `exportTransformedData()`**: Confirmed. The function opens a file per resource type per batch via `MustOutFile()` and never closes it. With 10 resource types enabled and a batch size of 64, processing 10,000 ledgers (~156 batches) leaks ~1,560 FDs from `outFile` alone.

2. **`createOutputFile()` leaks an additional FD**: Confirmed. When the file doesn't exist (the common case since filenames encode batch ranges), `os.Create()` opens a new FD and the return value is discarded. This doubles the leak to ~3,120 FDs for the same scenario.

3. **`UploadTo()` leaks a third FD**: The `reader` opened via `os.Open(path)` on line 32 is never closed. When GCS upload is enabled, each resource file leaks 3 FDs total. `deleteLocalFiles(path)` unlinks the file from the filesystem but does not release open kernel FDs.

4. **Fatal exit on exhaustion**: When `MustOutFile()` fails (e.g., `os.OpenFile` returns EMFILE), it calls `cmdLogger.Fatal()` which terminates the process. All subsequent batches are lost, producing a partial export.

5. **Consistency with other commands**: All nine one-shot export commands call `outFile.Close()`. This is a Pattern 3 (export command consistency) deviation — the batch-processing command is the only one that omits it.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` (create if not present; alternatively `cmd/command_utils_test.go`)
- **Setup**: Call `exportTransformedData()` in a loop with dummy `transformedOutput` containing a single resource type, using a temporary directory for `folderPath`. Before the loop, record the process FD count via `/proc/self/fd` (Linux) or `lsof -p` (macOS).
- **Steps**: Run `exportTransformedData()` N times (e.g., 100) with distinct batch start/end values so each call creates a new file. After all iterations, count open FDs again.
- **Assertion**: Assert that the FD count after the loop exceeds the initial count by at least N (proving descriptors accumulate). For a stronger assertion, set `ulimit -n` to a low value (e.g., 128) and verify that the loop fatals before completing all iterations.
