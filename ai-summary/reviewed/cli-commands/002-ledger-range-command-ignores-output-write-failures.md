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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the `get_ledger_range_from_times` command from flag parsing through `MustOutFile` (which opens the file successfully) to the write block at lines 74-78. Confirmed that `outFile.Write(marshalled)`, `outFile.WriteString("\n")`, and `outFile.Close()` all discard their return values. Unlike the `ExportEntry` helper (covered by success/002), this code path doesn't even log write failures — they are completely silent. The command exits with code 0 regardless of write outcome.

### Code Paths Examined

- `cmd/get_ledger_range_from_times.go:26-82` — full Run function; lines 74-78 perform three fallible I/O operations with no error checking
- `cmd/command_utils.go:31-53` — `MustOutFile` opens the file with `O_RDWR|O_CREATE|O_TRUNC`; file creation succeeds but subsequent writes are unchecked
- `cmd/command_utils.go:55-86` — `ExportEntry` (for comparison) at least logs write errors; this code path bypasses it entirely

### Findings

1. **All three I/O return values discarded**: `Write()` returns `(int, error)`, `WriteString()` returns `(int, error)`, and `Close()` returns `error`. None are checked.
2. **No logging of failures**: Unlike `ExportEntry` which logs write errors (even though it swallows them), this code path produces no diagnostic output at all on write failure.
3. **`Close()` error ignored**: On some filesystems, buffered data is only flushed during `Close()`. Ignoring the `Close()` error means even data that appeared to be written may not persist.
4. **Distinct from ExportEntry bug**: The success finding 002-export-entry-swallows-write-errors covers the `ExportEntry` helper. This hypothesis covers a separate, independent code path that writes directly to the file without any helper.
5. **Downstream impact**: The range file is consumed by automation to determine which ledgers to export. A corrupt or empty range file combined with a successful exit code could cause downstream export jobs to fail or export the wrong range.

### PoC Guidance

- **Test file**: `cmd/get_ledger_range_from_times_test.go` (create new, or append to existing test file in `cmd/`)
- **Setup**: Create a `ledgerRange` struct with known start/end values, marshal it, then attempt to write to a closed `*os.File` obtained from `MustOutFile` (or use `os.Pipe` with the read end closed as in the ExportEntry PoC)
- **Steps**: (1) Open a temp file via `MustOutFile`, (2) close it to simulate I/O failure, (3) call `outFile.Write(marshalled)` and `outFile.WriteString("\n")` and `outFile.Close()`, (4) observe that no error is surfaced
- **Assertion**: Verify that `Write` and `WriteString` return non-nil errors that the production code silently discards, confirming the command would exit 0 despite write failure
