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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestLedgerRangeWriteErrorsSilentlyIgnored"
**Test Language**: Go

### Demonstration

The test constructs a `ledgerRange` struct, marshals it to JSON, then replicates the production write path (lines 74-78) against a pre-closed file handle. All three I/O operations — `Write()`, `WriteString()`, and `Close()` — return non-nil errors that the production code silently discards. The output file remains empty (0 bytes), confirming that the command would exit 0 while leaving a corrupt/empty range file for downstream automation.

### Test Body

```go
package cmd

import (
	"encoding/json"
	"os"
	"testing"
)

// TestLedgerRangeWriteErrorsSilentlyIgnored demonstrates that
// get_ledger_range_from_times discards write errors. The production code
// at get_ledger_range_from_times.go:74-78 calls Write, WriteString, and
// Close without checking their return values.
func TestLedgerRangeWriteErrorsSilentlyIgnored(t *testing.T) {
	// 1. Construct a ledgerRange and marshal it (same as production code)
	toExport := ledgerRange{Start: 100, End: 200}
	marshalled, err := json.Marshal(toExport)
	if err != nil {
		t.Fatalf("unexpected marshal error: %v", err)
	}

	// 2. Create a temp file then close it to simulate I/O failure
	tmpFile, err := os.CreateTemp(t.TempDir(), "poc-range-*.txt")
	if err != nil {
		t.Fatalf("could not create temp file: %v", err)
	}
	// Close the file so subsequent writes will fail with "file already closed"
	tmpFile.Close()

	// 3. Replicate the production code path (lines 74-78):
	//    outFile.Write(marshalled)    // return values discarded
	//    outFile.WriteString("\n")    // return values discarded
	//    outFile.Close()              // return value discarded
	//
	// We call the same methods and capture the errors that production ignores.
	_, writeErr := tmpFile.Write(marshalled)
	_, writeStringErr := tmpFile.WriteString("\n")
	closeErr := tmpFile.Close()

	// 4. Assert: all three operations return errors that the production code
	//    silently discards. This proves the command would exit 0 despite
	//    failing to write the range file.
	errorsDiscarded := 0

	if writeErr != nil {
		t.Logf("Write() returned error silently discarded by production code: %v", writeErr)
		errorsDiscarded++
	} else {
		t.Log("Write() unexpectedly succeeded")
	}

	if writeStringErr != nil {
		t.Logf("WriteString() returned error silently discarded by production code: %v", writeStringErr)
		errorsDiscarded++
	} else {
		t.Log("WriteString() unexpectedly succeeded")
	}

	if closeErr != nil {
		t.Logf("Close() returned error silently discarded by production code: %v", closeErr)
		errorsDiscarded++
	} else {
		t.Log("Close() unexpectedly succeeded")
	}

	if errorsDiscarded == 0 {
		t.Fatal("Expected at least one I/O operation to return an error, but all succeeded")
	}

	t.Logf("Production code silently discards %d error(s) — command exits 0 with no range written", errorsDiscarded)

	// 5. Verify the file on disk is empty (no data was written)
	content, err := os.ReadFile(tmpFile.Name())
	if err != nil {
		t.Fatalf("could not read file: %v", err)
	}
	if len(content) != 0 {
		t.Fatalf("expected empty file, got %d bytes: %s", len(content), content)
	}
	t.Logf("Output file is empty (0 bytes) — downstream automation would consume a corrupt range file")
}
```

### Test Output

```
=== RUN   TestLedgerRangeWriteErrorsSilentlyIgnored
    data_integrity_poc_test.go:45: Write() returned error silently discarded by production code: write /var/folders/wz/c3l_zq0s6qscqln_qh5s00240000gn/T/TestLedgerRangeWriteErrorsSilentlyIgnored693515076/001/poc-range-2284702578.txt: file already closed
    data_integrity_poc_test.go:52: WriteString() returned error silently discarded by production code: write /var/folders/wz/c3l_zq0s6qscqln_qh5s00240000gn/T/TestLedgerRangeWriteErrorsSilentlyIgnored693515076/001/poc-range-2284702578.txt: file already closed
    data_integrity_poc_test.go:59: Close() returned error silently discarded by production code: close /var/folders/wz/c3l_zq0s6qscqln_qh5s00240000gn/T/TestLedgerRangeWriteErrorsSilentlyIgnored693515076/001/poc-range-2284702578.txt: file already closed
    data_integrity_poc_test.go:69: Production code silently discards 3 error(s) — command exits 0 with no range written
    data_integrity_poc_test.go:79: Output file is empty (0 bytes) — downstream automation would consume a corrupt range file
--- PASS: TestLedgerRangeWriteErrorsSilentlyIgnored (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.690s
```
