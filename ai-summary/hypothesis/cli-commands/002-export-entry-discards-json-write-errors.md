# H002: ExportEntry reports success after partial or failed JSON writes

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If writing a JSON row or its trailing newline fails, the export command should receive a non-nil error, count the row as failed, and avoid treating the output file as complete. Downstream upload should not proceed as though the exported dataset were intact.

## Mechanism

`ExportEntry` logs errors from `outFile.Write(...)` and `outFile.WriteString("\n")` but always returns `nil` to its callers. Every export command uses that helper to decide whether a row succeeded, so a short write, newline failure, or disk-full condition produces a truncated JSON file while the command still increments written-byte counters, reports successful transforms, and may upload the incomplete file to cloud storage.

## Trigger

Run any export command to a filesystem that starts returning write errors after some rows have been emitted (for example, a nearly full disk or a destination that injects short-write failures). The command continues past the failed write because `ExportEntry` returns `nil`, leaving a JSON file with missing or malformed line-delimited rows that still looks like a completed export.

## Target Code

- `cmd/command_utils.go:55-86` — write and newline errors are logged but not returned
- `cmd/export_ledgers.go:50-56` — callers trust `ExportEntry` to signal row failure
- `cmd/export_transactions.go:43-49` — same success accounting path for transactions
- `cmd/export_operations.go:43-49` — same success accounting path for operations
- `cmd/export_ledger_entry_changes.go:315-319` — batch exporter also treats `ExportEntry` as authoritative

## Evidence

The helper has signature `(int, error)` but returns `numBytes + newLineNumBytes, nil` even after either write call fails. Because all export commands gate their failure accounting on `if err != nil`, any I/O failure inside `ExportEntry` becomes invisible to command-level control flow.

## Anti-Evidence

The normal ledger-content path is unlikely to trigger marshal failures because the exported structs are regular Go values. This hypothesis depends on real write-path I/O failures, not on malformed Stellar data itself.
