# H004: `export_ledger_entry_changes` leaks JSON file descriptors until later batches stop exporting

**Date**: 2026-04-10
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each resource file opened during `export_ledger_entry_changes` should be closed once its batch has been written and, if configured, uploaded. Long-running or continuous exports should be able to process arbitrarily many batches without exhausting OS file descriptors or dropping later ledger-entry-change outputs.

## Mechanism

`exportTransformedData()` opens a new JSON file with `MustOutFile()` for every resource in every batch, writes records, and never calls `outFile.Close()`. In bounded ranges this may go unnoticed, but in large ranges or `end-ledger=0` streaming mode the command accumulates open descriptors until `os.OpenFile` fails, at which point later batches are never exported.

## Trigger

Run `export_ledger_entry_changes` across a long range or continuously with multiple resource types enabled. After enough batches/resources, the process should hit the host's open-file limit and stop producing later JSON exports even though earlier batches looked healthy.

## Target Code

- `cmd/export_ledger_entry_changes.go:304-372` — opens one `outFile` per resource and never closes it
- `cmd/command_utils.go:31-53` — `MustOutFile()` creates a real OS file descriptor each time

## Evidence

All the other export commands explicitly call `outFile.Close()` before upload or exit, but `exportTransformedData()` does not. The loop is batch-oriented and can run indefinitely, so the missing close compounds over time rather than being a one-off leak.

## Anti-Evidence

Short, bounded exports may finish before hitting the descriptor limit, so the failure is workload-dependent. The bug is still meaningful because it drops later batches rather than corrupting just a single row.
