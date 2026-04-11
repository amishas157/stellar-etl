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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`ExportEntry()` at `cmd/command_utils.go:55-86` performs two file I/O operations: `outFile.Write(marshalled)` at line 78 and `outFile.WriteString("\n")` at line 82. Both capture the error return, log it via `cmdLogger.Errorf`, then fall through to line 86 which unconditionally returns `nil`. All 10 export commands (ledgers, transactions, operations, effects, trades, assets, contract_events, ledger_entry_changes, token_transfers, ledger_transaction) call `ExportEntry()` and check its error return to decide whether to increment `numFailures` and whether to proceed with `MaybeUpload()`. Since I/O errors never surface, callers always treat writes as successful.

### Code Paths Examined

- `cmd/command_utils.go:ExportEntry:78-86` — `outFile.Write()` error captured at line 79, logged at line 80, then execution falls through; `outFile.WriteString("\n")` error captured at line 83, logged at line 84, then execution falls through; line 86 returns `numBytes + newLineNumBytes, nil` unconditionally
- `cmd/export_ledgers.go:50-55` — checks `err` from `ExportEntry()`, increments `numFailures` and `continue`s on error; since error is always nil, failure count is never incremented for I/O failures
- `cmd/export_transactions.go:43-48` — identical pattern: error check gates `numFailures` increment
- `cmd/export_operations.go:43` — same pattern
- `cmd/export_effects.go:45` — same pattern
- `cmd/export_trades.go:47` — same pattern
- `cmd/export_assets.go:59` — same pattern
- `cmd/export_token_transfers.go:47` — same pattern
- `cmd/export_contract_events.go:43` — same pattern
- `cmd/export_ledger_transaction.go:42` — same pattern
- `cmd/export_ledger_entry_changes.go:316-319` — returns error directly to caller; since nil, batch loop continues and upload proceeds

### Findings

The bug is confirmed. `ExportEntry()` has a function signature `(int, error)` but only returns a non-nil error from the second `json.Marshal()` call (line 73-76). The two file I/O operations at lines 78-85 both capture errors but only log them. This creates two concrete failure modes:

1. **Truncated JSON rows**: If `outFile.Write(marshalled)` partially writes (short write) or fails entirely, the output file contains an incomplete or missing JSON line. The caller does not know and does not increment its failure counter.

2. **Missing newlines**: If `outFile.WriteString("\n")` fails after a successful `Write`, the file contains concatenated JSON objects without line delimiters, making downstream line-delimited JSON parsers misparse multiple rows.

In both cases, `MaybeUpload()` is subsequently called on the malformed file, and `PrintTransformStats()` reports 0 failures for what may be many corrupted rows. The upload to GCS succeeds with bad data.

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (create if not present; or append to an existing cmd-level test file)
- **Setup**: Create a mock `*os.File` or use a real file on a filesystem where writes can be made to fail. The simplest approach: create an `os.File` wrapper that returns `os.ErrClosed` from `Write`/`WriteString`, or close the file handle before calling `ExportEntry()`.
- **Steps**:
  1. Create a valid transform output struct (e.g., `transform.LedgerOutput{}`)
  2. Open a file with `MustOutFile()`, then close it immediately (`outFile.Close()`)
  3. Call `ExportEntry(transformed, outFile, nil)`
  4. Observe the returned error
- **Assertion**: Assert that the returned `error` is non-nil when `Write` fails. Currently it will be `nil`, demonstrating the bug. Also assert that the returned `numBytes` does not include bytes from the failed write.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportEntrySwallowsWriteError"
**Test Language**: Go

### Demonstration

The test calls `ExportEntry()` with a closed file handle, causing both `outFile.Write()` and `outFile.WriteString("\n")` to fail with "file already closed" errors. Despite both I/O operations failing, `ExportEntry()` returns `nil` error and `numBytes=0`. This proves that all 10 export commands would treat a failed write as successful, skip incrementing their failure counter, and proceed to upload a malformed or empty file via `MaybeUpload()`.

### Test Body

```go
package cmd

import (
	"os"
	"testing"
)

// TestExportEntrySwallowsWriteError demonstrates that ExportEntry() returns nil
// error even when the underlying file write fails, because it logs I/O errors
// instead of propagating them.
func TestExportEntrySwallowsWriteError(t *testing.T) {
	// 1. Create a temporary file, then close it to make writes fail
	tmpFile, err := os.CreateTemp(t.TempDir(), "poc-*.jsonl")
	if err != nil {
		t.Fatalf("Failed to create temp file: %v", err)
	}
	filePath := tmpFile.Name()
	tmpFile.Close() // Close the file handle — subsequent writes will fail

	// Re-open so we have an *os.File value, then close the fd underneath
	closedFile, err := os.OpenFile(filePath, os.O_RDWR, 0644)
	if err != nil {
		t.Fatalf("Failed to reopen temp file: %v", err)
	}
	closedFile.Close() // Now closedFile is an *os.File with a closed fd

	// 2. Build a simple serializable entry
	entry := map[string]string{"key": "value"}

	// 3. Call ExportEntry with the closed file — Write will fail
	numBytes, exportErr := ExportEntry(entry, closedFile, nil)

	// 4. Demonstrate the bug: ExportEntry returns nil error despite write failure
	if exportErr != nil {
		t.Fatalf("ExportEntry returned non-nil error (bug may be fixed): %v", exportErr)
	}
	t.Logf("BUG CONFIRMED: ExportEntry returned nil error despite write to closed file")
	t.Logf("Returned numBytes=%d (includes bytes from failed writes)", numBytes)

	// The correct behavior would be to return a non-nil error, but ExportEntry
	// swallows I/O errors at lines 78-85 of command_utils.go and always
	// returns nil at line 86.

	// Verify the file is empty (write did not succeed)
	info, err := os.Stat(filePath)
	if err != nil {
		t.Fatalf("Could not stat file: %v", err)
	}
	if info.Size() != 0 {
		t.Fatalf("Expected empty file after failed write, got %d bytes", info.Size())
	}
	t.Logf("File is empty (0 bytes) — write failed but ExportEntry reported success")
	t.Logf("A caller checking this error would proceed to MaybeUpload() with bad data")
}
```

### Test Output

```
=== RUN   TestExportEntrySwallowsWriteError
time="2026-04-11T12:05:03.647-05:00" level=error msg="Error writing map[key:value] to file: write /var/folders/wz/c3l_zq0s6qscqln_qh5s00240000gn/T/TestExportEntrySwallowsWriteError2689954767/001/poc-1846210504.jsonl: file already closed" pid=51544
time="2026-04-11T12:05:03.648-05:00" level=error msg="Error writing new line to file /var/folders/wz/c3l_zq0s6qscqln_qh5s00240000gn/T/TestExportEntrySwallowsWriteError2689954767/001/poc-1846210504.jsonl: file already closed" pid=51544
    data_integrity_poc_test.go:37: BUG CONFIRMED: ExportEntry returned nil error despite write to closed file
    data_integrity_poc_test.go:38: Returned numBytes=0 (includes bytes from failed writes)
    data_integrity_poc_test.go:52: File is empty (0 bytes) — write failed but ExportEntry reported success
    data_integrity_poc_test.go:53: A caller checking this error would proceed to MaybeUpload() with bad data
--- PASS: TestExportEntrySwallowsWriteError (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.749s
```
