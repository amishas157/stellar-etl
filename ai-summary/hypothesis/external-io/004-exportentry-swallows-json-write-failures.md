# H004: `ExportEntry()` hides JSON write failures behind success-shaped returns

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If writing an exported JSON row fails, the export command should receive a non-nil error, count the row as failed, and avoid reporting a successful export of a partially written file.

## Mechanism

`ExportEntry()` logs errors from `outFile.Write()` and `outFile.WriteString("\n")` but always returns `numBytes + newLineNumBytes, nil`. Every JSON export command relies on that error return to decide whether to increment failure counts and whether the output file is valid enough to upload, so partial writes can silently leave truncated JSON lines or missing newlines while the CLI reports success and proceeds to `MaybeUpload()`.

## Trigger

Run any JSON export (`export_transactions`, `export_ledgers`, `export_ledger_entry_changes`, etc.) on a filesystem that returns a short write or write error mid-export, such as a full disk or unstable network mount. The command should fail that row, but this code path should keep going and may upload a malformed line-delimited JSON file.

## Target Code

- `cmd/command_utils.go:55-86` - logs `Write`/`WriteString` errors but returns `nil`
- `cmd/export_ledgers.go:50-73` - representative caller that trusts `ExportEntry()` errors for failure accounting
- `cmd/export_transactions.go:43-66` - representative caller that trusts `ExportEntry()` errors for failure accounting
- `cmd/export_ledger_entry_changes.go:311-372` - batch exporter also treats a nil `ExportEntry()` return as success

## Evidence

The only non-nil return in `ExportEntry()` is the later `json.Marshal(i)` failure path. Actual file I/O failures at lines 78-85 are merely logged, after which the function still returns success to all callers.

## Anti-Evidence

Pure marshalling failures are surfaced correctly because the second `json.Marshal(i)` return is propagated. On healthy local disks, the bug stays latent because `Write` and `WriteString` usually succeed.
