# H005: `ExportEntry` reports success after JSON write failures

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Silent partial exports after I/O errors
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If writing the JSON row or trailing newline fails, `ExportEntry()` should return that error so the caller can fail the row, abort the export, or skip cloud upload. Byte counters should only include bytes that were successfully persisted.

## Mechanism

`ExportEntry()` logs errors from `outFile.Write()` and `WriteString("\n")` but always returns `nil` anyway. Every caller therefore treats short writes, `ENOSPC`, and other I/O failures as successful rows, increments exported byte counts, and can proceed to upload a truncated file that looks complete except for the silently dropped records.

## Trigger

Cause any JSON export command to hit a filesystem write error while writing rows, for example by exhausting disk space or encountering an I/O error on the output path.

## Target Code

- `cmd/command_utils.go:55-86` — `ExportEntry()` logs write errors but still returns `nil`
- `cmd/export_transactions.go:43-49` — representative caller that trusts the returned error to detect export failure

## Evidence

The only write path that returns a non-nil error is the final `json.Marshal(i)`; both file writes ignore `err` after logging. The function also returns `numBytes + newLineNumBytes` even when one or both writes failed.

## Anti-Evidence

On healthy local disks this stays invisible, so ordinary regression tests may not catch it. Callers do check returned errors correctly, but they never receive one for the most important I/O failures because `ExportEntry()` swallows them.
