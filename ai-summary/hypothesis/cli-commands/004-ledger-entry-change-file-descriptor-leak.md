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
