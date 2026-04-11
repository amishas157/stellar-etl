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
