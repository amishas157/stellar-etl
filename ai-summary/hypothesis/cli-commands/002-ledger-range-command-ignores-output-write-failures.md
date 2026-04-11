# H002: `get_ledger_range_from_times` reports success after partial file writes

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `get_ledger_range_from_times` writes its JSON range to `--output`, it should return an error if either the JSON bytes or the trailing newline cannot be written completely. The command should not leave a truncated range file that looks like a completed export.

## Mechanism

Unlike the shared export commands, `get_ledger_range_from_times` writes directly to `outFile` and discards both write return values. A short write, disk-full condition, or injected write failure can therefore leave a malformed `{"start":...,"end":...}` file on disk while the command exits as though the range was written successfully.

## Trigger

Run `get_ledger_range_from_times -o <path>` on a filesystem that starts returning short writes or write errors after the file is opened. The command still reaches `outFile.Close()` without surfacing the failed write, leaving a partially written JSON range file for downstream automation to consume.

## Target Code

- `cmd/get_ledger_range_from_times.go:68-78` — marshals the range, writes it, and ignores both write errors and byte counts

## Evidence

The code calls `outFile.Write(marshalled)` and `outFile.WriteString("\n")` without assigning either returned `(n, err)` pair to variables. There is no surrounding helper like `ExportEntry` and no follow-up validation of the number of bytes written before the file is closed.

## Anti-Evidence

If the command writes to stdout (`-o ""`), this file-path corruption path is not exercised. On ordinary local filesystems the writes usually succeed, so the bug is most visible under I/O error conditions rather than ordinary ledger-range lookups.
