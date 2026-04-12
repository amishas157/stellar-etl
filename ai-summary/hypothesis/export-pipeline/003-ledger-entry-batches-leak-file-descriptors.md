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
