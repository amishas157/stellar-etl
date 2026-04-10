# H004: Continuous ledger-entry-change exports eventually stop producing batches due to leaked file descriptors

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledger_entry_changes` should be able to run for long periods in continuous mode, closing each batch's output files after writing and uploading them so later ledger ranges continue exporting normally. The correct result is one JSON file per selected resource per batch, for the full runtime of the process.

## Mechanism

Two independent leaks compound here. `createOutputFile` creates missing files with `os.Create` and never closes that descriptor, and `exportTransformedData` opens each per-resource output file with `MustOutFile(path)` but never closes it before moving to the next resource or next batch. In continuous mode with upload enabled, `UploadTo` deletes each local file after upload, forcing the next batch to recreate and reopen fresh paths; over time, the process consumes the descriptor limit and later batches stop exporting entirely.

## Trigger

1. Run `export_ledger_entry_changes` without `--end-ledger` so it streams indefinitely, preferably with several export flags enabled.
2. Enable cloud upload so each batch deletes its local files after upload, causing the next batch to recreate them.
3. After enough batches, `os.Create` or `os.OpenFile` starts failing, and subsequent ledger ranges are missing from output because the command can no longer open batch files.

## Target Code

- `cmd/command_utils.go:createOutputFile:19-29` — calls `os.Create(filepath)` and discards the returned file without closing it
- `cmd/command_utils.go:MustOutFile:42-52` — always reopens the same path after `createOutputFile`
- `cmd/export_ledger_entry_changes.go:304-372` — opens `outFile := MustOutFile(path)` for each resource and never calls `outFile.Close()`
- `cmd/upload_to_gcs.go:71` — deletes local files after upload, ensuring the next batch recreates them

## Evidence

Unlike the one-shot exporters, `export_ledger_entry_changes` writes files inside a long-running per-batch loop and omits any `outFile.Close()` call in that loop. Combined with the leaked descriptor in `createOutputFile`, continuous exports recreate and reopen output paths over and over without releasing prior descriptors.

## Anti-Evidence

Short bounded exports may complete before hitting the process file limit, so the bug is easy to miss in normal tests. The loss appears only after enough batches accumulate, which fits the subsystem's continuous-streaming mode much more than its one-off batch commands.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `exportTransformedData` (export_ledger_entry_changes.go:295-377) and confirmed that `outFile` opened on line 311 via `MustOutFile(path)` is never closed. Compared all 10 export commands: all 9 siblings (`export_ledgers.go:63`, `export_transactions.go:56`, `export_operations.go:56`, `export_effects.go:59`, `export_trades.go:61`, `export_assets.go:72`, `export_contract_events.go:57`, `export_token_transfers.go:57`, `export_ledger_transaction.go:51`) call `outFile.Close()` after writing. `export_ledger_entry_changes` is the sole outlier. Additionally confirmed that `createOutputFile` (command_utils.go:22) discards the `*os.File` from `os.Create` without closing it.

### Code Paths Examined

- `cmd/command_utils.go:createOutputFile:19-29` — `os.Create(filepath)` return value assigned to `_`, fd leaked whenever file doesn't exist
- `cmd/command_utils.go:MustOutFile:31-53` — calls `createOutputFile` (leaks 1 fd) then `os.OpenFile` (returns 2nd fd to caller)
- `cmd/export_ledger_entry_changes.go:304-377` (`exportTransformedData`) — `outFile := MustOutFile(path)` on line 311, used for writes in inner loop, never closed; `MaybeUpload` on line 368 may delete the file while the fd is still open
- `cmd/export_ledger_entry_changes.go:79-291` (outer batch loop) — calls `exportTransformedData` for each batch indefinitely in continuous mode
- `cmd/export_ledgers.go:37,63` — reference pattern showing correct `outFile.Close()` call
- `cmd/upload_to_gcs.go:32,71` — `os.Open(path)` for upload also never closed (secondary leak), then `deleteLocalFiles` removes path

### Findings

**Bug 1 — `createOutputFile` fd leak (command_utils.go:22)**: `os.Create(filepath)` returns a `*os.File` that is discarded to `_`. The file descriptor is never closed. Since batch filenames embed start-end ledger numbers (`exportFilename` returns `%d-%d-%s.txt`), every batch creates unique paths, so this branch is taken every call — leaking 1 fd per resource per batch.

**Bug 2 — Missing `outFile.Close()` in `exportTransformedData` (export_ledger_entry_changes.go:311)**: This is a classic Pattern 3 outlier. All 9 sibling export commands call `outFile.Close()` after writing. `exportTransformedData` does not. Each iteration of the `for resource, output := range` loop opens a new file handle that is never explicitly closed — leaking 1 fd per resource per batch.

**Combined impact**: With N resource types enabled (up to 10), each batch leaks 2N file descriptors. At 20 fds/batch and a typical ulimit of 1024, exhaustion occurs after ~50 batches. When exhausted, `MustOutFile` calls `cmdLogger.Fatal` causing a crash (not silent loss).

**Correction to hypothesis**: The upload/delete cycle is not necessary to trigger the leak. Batch filenames are always unique (different ledger ranges), so `createOutputFile` always hits the `os.IsNotExist` branch regardless of upload. Also, exhaustion causes a Fatal crash rather than silent data loss. Go's GC finalizers may reclaim some fds nondeterministically, making the failure intermittent rather than guaranteed.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` (or create a new test file)
- **Setup**: Call `exportTransformedData` in a loop with mock data, using a small `ulimit -n` or counting open fds via `/proc/self/fd` (Linux) or `lsof` (macOS)
- **Steps**: (1) Create a transformedOutput map with several resource types, each containing a small slice of mock data. (2) Call `exportTransformedData` in a loop for 100+ iterations with unique start/end values. (3) After each iteration, count open file descriptors for the process.
- **Assertion**: Assert that open fd count grows linearly with iterations (confirming the leak). Alternatively, assert that after adding `outFile.Close()` to `exportTransformedData` and fixing `createOutputFile` to close its handle, the fd count remains stable across iterations.
