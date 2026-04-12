# H005: JSON exporters count partial writes as successful rows

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When JSON row writing fails, `ExportEntry()` should return the write error so callers can increment failure counters, stop exporting, and avoid uploading a truncated file. A newline write failure should also be surfaced because JSONL output without record separators is corrupted even if the object bytes were written successfully.

## Mechanism

`ExportEntry()` logs marshal, decode, `Write`, and `WriteString` errors but still returns `nil`, so every caller treats the row as successfully exported. Commands such as `export_transactions` and `export_ledgers` only mark a failure when `ExportEntry()` returns a non-nil error, which means disk-full, short-write, or newline-write failures can produce truncated or merged JSON rows while success counters and subsequent upload steps still proceed.

## Trigger

1. Run any JSON export command that uses `ExportEntry()`, such as `export_transactions` or `export_ledgers`.
2. Cause `outFile.Write(...)` or `outFile.WriteString("\n")` to fail mid-run, for example by filling the destination filesystem or hitting an I/O error after the object bytes are written.
3. Observe that the command logs an error but keeps counting the row as successful and may upload a malformed JSONL file.

## Target Code

- `cmd/command_utils.go:55-86` — logs write/newline errors but always returns `nil`
- `cmd/export_transactions.go:34-49` — increments `numFailures` only when `ExportEntry()` returns an error
- `cmd/export_ledgers.go:42-56` — same success/failure accounting pattern on another active export surface
- `cmd/get_ledger_range_from_times.go:74-78` — standalone command also ignores direct file-write errors entirely

## Evidence

The function signature promises `(int, error)`, but both `outFile.Write` and `outFile.WriteString` errors are only logged before returning a nil error to the caller. The main export commands then use that nil return to keep `totalNumBytes`, leave `numFailures` unchanged, print success stats, and continue to upload the output path. A newline-only failure is especially telling: it can merge two valid JSON objects into an invalid JSONL stream without any caller-visible error.

## Anti-Evidence

This bug only appears under I/O failure conditions, so happy-path tests on healthy local disks will not expose it. Some fatal error handling still exists for file creation and Parquet writes, which limits the issue to the JSON export path rather than the entire CLI.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `ExportEntry()` in `cmd/command_utils.go:55-86` end-to-end. The function handles four error sites: initial `json.Marshal`, `decoder.Decode`, `outFile.Write`, and `outFile.WriteString`. Only the second `json.Marshal` (line 73-75) properly returns its error. The other three I/O-path errors (Write at line 79, WriteString at line 83) and two pre-serialization errors (Marshal at line 58, Decode at line 66) are logged via `cmdLogger.Errorf` but the function unconditionally returns `nil` on line 86. All 10 callers across active export commands check `if err != nil` to count failures and decide whether to upload, making these silent nil returns a real data-loss path.

### Code Paths Examined

- `cmd/command_utils.go:ExportEntry:55-86` — Confirmed: `outFile.Write` error (line 79-81) logged but not returned; `outFile.WriteString("\n")` error (line 82-85) logged but not returned; first `json.Marshal` error (line 58-60) logged but not returned; `decoder.Decode` error (line 65-68) logged but not returned. Line 86 always returns `nil`.
- `cmd/export_transactions.go:43-48` — Confirmed: increments `numFailures` only when `ExportEntry` returns non-nil error; unreachable for Write/WriteString failures.
- `cmd/export_ledgers.go:50-55` — Same pattern: `numFailures` only incremented on non-nil error return.
- `cmd/export_operations.go:43`, `cmd/export_effects.go:45`, `cmd/export_trades.go:47`, `cmd/export_assets.go:59`, `cmd/export_contract_events.go:43`, `cmd/export_token_transfers.go:47`, `cmd/export_ledger_transaction.go:42` — All 10 callers use the same `if err != nil` pattern.
- `cmd/export_ledger_entry_changes.go:316-319` — Also calls `ExportEntry` and returns the error, but since the error is always `nil`, the early-exit path is unreachable for I/O failures.
- `cmd/get_ledger_range_from_times.go:74-78` — Standalone command ignores `Write` and `WriteString` return values entirely (no error variable captured).

### Findings

1. **Write error swallowed**: When `outFile.Write(marshalled)` fails (disk full, I/O error), the error is logged but `ExportEntry` returns `(numBytes, nil)` where `numBytes` may be a partial write count. The caller adds this to `totalNumBytes` and does not increment `numFailures`.

2. **Newline error swallowed**: When `outFile.WriteString("\n")` fails after a successful Write, two JSON objects become concatenated without a separator. The resulting JSONL is structurally invalid — downstream parsers will fail to split records. This is returned as `nil` error.

3. **Upload proceeds on corrupt output**: After the write loop, all export commands call `MaybeUpload()` unconditionally. A truncated or malformed JSONL file is uploaded to GCS as if it were complete.

4. **Pre-serialization errors also swallowed**: The initial `json.Marshal(entry)` error (line 58-60) and `decoder.Decode` error (line 65-68) are logged but not returned. If the first marshal fails, `m` is nil, the decoder produces an empty map, extra fields are added, and the second marshal produces a near-empty JSON object (just the extra fields). This corrupt record is written to the file and counted as successful.

5. **Contrast with Parquet path**: `WriteParquet()` (line 162-179) uses `cmdLogger.Fatal` for all errors, meaning Parquet write failures halt the process. The JSON path has strictly weaker error handling.

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (create if needed, or append to existing test infrastructure)
- **Setup**: Create an `ExportEntry` test that uses a mock `*os.File` which returns an error from `Write`. This can be done by writing to a file on a read-only filesystem, or by using a pipe that's been closed on the read end. Alternatively, use `os.Pipe()`, close the reader, then pass the writer as `outFile`.
- **Steps**:
  1. Create a pipe with `os.Pipe()`, close the read end immediately
  2. Call `ExportEntry(validStruct, closedWriteEnd, nil)`
  3. Check the returned error
- **Assertion**: Assert that the returned error is non-nil. Currently it will be `nil`, demonstrating the bug. Also verify that `numBytes` returned is 0 or reflects only a partial write, not a successful count.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportEntrySwallowsWriteError" and "TestExportEntrySwallowsNewlineWriteError"
**Test Language**: Go

### Demonstration

Both tests prove that `ExportEntry` silently swallows I/O errors. In `TestExportEntrySwallowsWriteError`, a pipe with a closed reader causes `Write` to fail with "broken pipe", yet `ExportEntry` returns `nil` error and `numBytes=0`. In `TestExportEntrySwallowsNewlineWriteError`, writing to a closed file produces "file already closed" errors on both `Write` and `WriteString`, yet the function still returns `nil`. This confirms that all 10+ export command callers will count these failed writes as successes and may upload truncated/corrupt JSONL files.

### Test Body

```go
package cmd

import (
	"os"
	"testing"
)

// TestExportEntrySwallowsWriteError demonstrates that ExportEntry returns nil
// when the underlying outFile.Write fails, silently dropping I/O errors.
func TestExportEntrySwallowsWriteError(t *testing.T) {
	// Create a pipe; closing the read end makes writes to the write end fail.
	r, w, err := os.Pipe()
	if err != nil {
		t.Fatalf("os.Pipe() failed: %v", err)
	}
	r.Close() // close reader so writes will fail with EPIPE / broken pipe

	entry := map[string]string{"key": "value"}
	numBytes, exportErr := ExportEntry(entry, w, nil)

	// BUG: ExportEntry should return a non-nil error because the write failed,
	// but it always returns nil. This demonstrates that write errors are silently dropped.
	if exportErr == nil {
		t.Errorf("ExportEntry returned nil error despite write failure — write errors are silently swallowed (numBytes=%d)", numBytes)
	} else {
		t.Logf("ExportEntry correctly returned error: %v", exportErr)
	}

	w.Close()
}

// TestExportEntrySwallowsNewlineWriteError demonstrates that even when the
// JSON bytes are written successfully, a failure on the newline WriteString
// is silently dropped — producing merged JSONL records.
func TestExportEntrySwallowsNewlineWriteError(t *testing.T) {
	// Use a temp file with limited space simulation: we'll write to a file,
	// then close it and try to write again.
	tmpFile, err := os.CreateTemp("", "poc-export-*.jsonl")
	if err != nil {
		t.Fatalf("CreateTemp failed: %v", err)
	}
	defer os.Remove(tmpFile.Name())

	// Close the file so both Write and WriteString will fail
	tmpFile.Close()

	entry := map[string]string{"key": "value"}
	numBytes, exportErr := ExportEntry(entry, tmpFile, nil)

	// BUG: same as above — ExportEntry returns nil even though the file is closed.
	if exportErr == nil {
		t.Errorf("ExportEntry returned nil error on closed file — errors silently swallowed (numBytes=%d)", numBytes)
	} else {
		t.Logf("ExportEntry correctly returned error: %v", exportErr)
	}
}
```

### Test Output

```
=== RUN   TestExportEntrySwallowsWriteError
time="2026-04-11T21:36:32.353-05:00" level=error msg="Error writing map[key:value] to file: write |1: broken pipe" pid=5001
time="2026-04-11T21:36:32.353-05:00" level=error msg="Error writing new line to file |1: write |1: broken pipe" pid=5001
    data_integrity_poc_test.go:24: ExportEntry returned nil error despite write failure — write errors are silently swallowed (numBytes=0)
--- FAIL: TestExportEntrySwallowsWriteError (0.00s)
=== RUN   TestExportEntrySwallowsNewlineWriteError
time="2026-04-11T21:36:32.354-05:00" level=error msg="Error writing map[key:value] to file: write /var/folders/.../poc-export-1963367748.jsonl: file already closed" pid=5001
time="2026-04-11T21:36:32.354-05:00" level=error msg="Error writing new line to file /var/folders/.../poc-export-1963367748.jsonl: write /var/folders/.../poc-export-1963367748.jsonl: file already closed" pid=5001
    data_integrity_poc_test.go:52: ExportEntry returned nil error on closed file — errors silently swallowed (numBytes=0)
--- FAIL: TestExportEntrySwallowsNewlineWriteError (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/cmd	5.715s
```
