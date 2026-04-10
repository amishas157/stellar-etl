# H002: ExportEntry counts failed JSON writes as successful exports

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a transformed row cannot be written to the JSON output file, the export command should receive a non-nil error, count that row as failed, and avoid reporting or uploading a silently truncated export. The output file should either contain every successfully transformed row or the command should fail explicitly.

## Mechanism

`ExportEntry` logs errors from `json.Marshal`, `decoder.Decode`, `outFile.Write`, and `outFile.WriteString`, but it only returns a non-nil error for the final `json.Marshal(i)` call. Every caller in `cmd/` treats a nil return as success, so disk-full, short-write, or newline-write failures can produce missing or malformed rows while transform stats and follow-on upload logic still treat the export as successful.

## Trigger

1. Start any JSON export command that writes many rows, such as `export_transactions`, `export_operations`, `export_effects`, or `export_ledger_entry_changes`.
2. Hit a legitimate filesystem error mid-run, such as `ENOSPC` or `EIO`, during `outFile.Write` or `outFile.WriteString`.
3. The command logs an error but still increments byte counters, keeps processing later rows, and can upload a truncated file as if the batch were complete.

## Target Code

- `cmd/command_utils.go:ExportEntry:55-86` — logs write and decode errors but returns `nil`
- `cmd/export_transactions.go:Run:43-49` — treats `ExportEntry` nil as a successful row
- `cmd/export_operations.go:Run:43-49` — same success accounting pattern
- `cmd/export_effects.go:Run:45-51` — same success accounting pattern
- `cmd/export_ledger_entry_changes.go:exportTransformedData:315-319` — same success accounting pattern before upload

## Evidence

The write path has three separate logged error branches (`json.Marshal(entry)`, `outFile.Write`, `outFile.WriteString`) that do not change the returned error. All callers rely on the returned error to increment `numFailures` or abort the batch, so these failures are observationally downgraded from "failed export" to "successful export with a log line."

## Anti-Evidence

The function does correctly fail if the second `json.Marshal(i)` call errors, and several callers are prepared to handle a non-nil return. The silent corruption only happens on the broader set of I/O and decode failures that `ExportEntry` currently suppresses.
