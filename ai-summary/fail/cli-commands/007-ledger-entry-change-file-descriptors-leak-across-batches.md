# H005: Ledger-entry-change exports leak file descriptors until later batches stop producing output

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each per-resource output file opened during `export_ledger_entry_changes` should be closed once its JSON rows are written and any optional upload is complete. Long-running or continuous exports should keep producing files for later batches instead of failing after accumulating open descriptors.

## Mechanism

`createOutputFile` calls `os.Create(filepath)` and never closes the returned descriptor, `MustOutFile` then opens the same path again, and `exportTransformedData` never closes its `outFile` at all. In bounded runs with many batches — or continuous mode with `end-ledger=0` — every resource file leaks descriptors permanently, so eventually later batches fail to open output or upload files and stop producing data for valid ledger ranges.

## Trigger

Run `export_ledger_entry_changes` for a long range or in continuous mode with several export flags enabled, especially with cloud upload so each resource produces a separate file per batch. After enough batch/resource combinations, the process approaches the OS open-file limit and later calls to `MustOutFile` or `os.Open` fail, causing subsequent batch outputs to be missing.

## Target Code

- `cmd/command_utils.go:19-26` — `createOutputFile` creates a file but never closes it
- `cmd/command_utils.go:31-52` — `MustOutFile` opens a second descriptor for the same path
- `cmd/export_ledger_entry_changes.go:309-316` — a new output file is opened for every resource in every batch
- `cmd/export_ledger_entry_changes.go:368-372` — upload occurs, but no `outFile.Close()` ever follows
- `cmd/upload_to_gcs.go:71-71` — uploaded files are deleted locally while leaked descriptors stay alive until process exit

## Evidence

Unlike the one-shot export commands, `export_ledger_entry_changes` loops forever over batches and resources, so repeated `MustOutFile` calls are on the hot path. There is no `defer outFile.Close()` or explicit close in `exportTransformedData`, making the leak cumulative rather than transient.

## Anti-Evidence

Short local runs may finish before exhausting the process's descriptor limit, which makes this easy to miss in existing tests. The bug becomes meaningful on sustained or high-cardinality exports where the command is expected to keep writing future batches.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of ai-summary/success/cli-commands/004-ledger-entry-changes-leaks-file-descriptors.md.gh-published
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same file descriptor leak that was already confirmed, PoC-tested, and published as success finding 004. The success finding documents the same root cause (`exportTransformedData` never closing `outFile` from `MustOutFile`, plus `createOutputFile` discarding the `*os.File` from `os.Create`), the same affected code paths, the same trigger scenario, and the same severity. A prior review (fail/006) also identified this as a duplicate.

### Code Paths Examined

- `cmd/command_utils.go:createOutputFile:19-28` — confirmed `os.Create()` return value discarded without close (identical to 004)
- `cmd/command_utils.go:MustOutFile:31-52` — confirmed returns `*os.File` that caller never closes (identical to 004)
- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-377` — confirmed `outFile` opened at line 311 is never closed (identical to 004)

### Why It Failed

This is a duplicate of the already-confirmed and published finding `ai-summary/success/cli-commands/004-ledger-entry-changes-leaks-file-descriptors.md.gh-published`. The success finding covers the identical root cause, identical affected code paths, identical trigger conditions, and identical severity assessment. Additionally, a prior review of this same hypothesis already exists at `ai-summary/fail/cli-commands/006-ledger-entry-change-file-descriptors-leak-across-batches.md`.

### Lesson Learned

The file descriptor leak in `export_ledger_entry_changes` has been thoroughly investigated, confirmed, and published. Hypothesis generators should check both the success directory and existing fail reviews before re-proposing known issues.
