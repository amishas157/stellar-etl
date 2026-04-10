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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced `ExportEntry()` in `cmd/command_utils.go:55-86` line by line. The function performs two `json.Marshal` calls (lines 57 and 73), a `json.Decode` (line 65), an `outFile.Write` (line 78), and an `outFile.WriteString` (line 82). Of these five error-producing operations, only the second `json.Marshal` (line 73-76) returns its error to the caller via `return 0, fmt.Errorf(...)`. The other four — including both file I/O operations — log the error and continue, ultimately returning `nil` on line 86. I confirmed all 10+ callers across every export command check `if err != nil` to detect failures, meaning none will ever learn about write errors.

### Code Paths Examined

- `cmd/command_utils.go:55-86` — `ExportEntry()`: confirmed lines 78-81 log `outFile.Write` error but do not return it; lines 82-85 log `outFile.WriteString("\n")` error but do not return it; line 86 unconditionally returns `nil`
- `cmd/command_utils.go:73-76` — the ONLY error path that returns non-nil: second `json.Marshal(i)` failure
- `cmd/export_transactions.go:43-48` — representative caller: checks `err != nil`, increments `numFailures`, continues. Will never trigger for write errors.
- `cmd/export_operations.go:43-46` — same pattern: checks err, increments failures, continues
- `cmd/export_ledgers.go:50-53` — same pattern
- `cmd/export_effects.go:45-48` — same pattern
- `cmd/export_trades.go:47-50` — same pattern
- `cmd/export_assets.go:59-62` — same pattern
- `cmd/export_contract_events.go:43-46` — same pattern
- `cmd/export_token_transfers.go:47-50` — same pattern
- `cmd/export_ledger_transaction.go:42-45` — same pattern
- `cmd/export_ledger_entry_changes.go:316-319` — most critical caller: returns err immediately to abort batch. But will never receive one for write failures, so batch processing continues through I/O errors.
- `cmd/command_utils.go:123-146` — `MaybeUpload()`: uploads file to GCS after export. Called after `ExportEntry` loop completes. If writes silently failed, the truncated file gets uploaded.

### Findings

The bug is confirmed and matches Investigation Pattern 6 (Error propagation). Specifically:

1. **Error swallowing on file write (line 78-81)**: `outFile.Write(marshalled)` error is logged via `cmdLogger.Errorf` but not returned. On ENOSPC or any I/O error, the row is silently dropped from the output file.

2. **Error swallowing on newline write (line 82-85)**: `outFile.WriteString("\n")` error is also logged and swallowed. If the data write succeeds but newline fails, the next row will be concatenated with the current one, producing malformed JSONL.

3. **Byte count inflation**: Line 86 returns `numBytes + newLineNumBytes` even after failures. When `Write` returns a partial byte count alongside an error, that partial count is still reported to the caller as successfully exported bytes.

4. **All callers affected**: Every export command (10 commands across 10 files) relies on `ExportEntry`'s error return to count failures and detect problems. None will ever detect a write-time I/O failure.

5. **GCS upload of truncated files**: After the export loop, commands call `MaybeUpload()` which uploads the output file. If write errors occurred mid-export, the uploaded file is missing rows but no error was raised to prevent the upload.

**Additional observation**: Lines 57-68 also swallow errors from the initial `json.Marshal(entry)` and `decoder.Decode(&i)`. If the first marshal fails, `m` is empty/nil, the decoder produces an error (also swallowed), `i` remains an empty map, and the second marshal on line 73 succeeds by producing `{}` (plus any extra fields). This writes a nearly-empty JSON object to the file — a more severe data corruption path, though unlikely in practice since marshalling well-typed structs rarely fails.

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (create if not present, or append to existing)
- **Setup**: Create a mock `*os.File` that returns an error on `Write()`. Alternatively, use a real file on a full filesystem (e.g., write to `/dev/full` on Linux) or open a file read-only.
- **Steps**: Call `ExportEntry(validStruct, brokenFile, nil)` where `brokenFile` is an `*os.File` whose Write method will fail.
- **Assertion**: Assert that the returned `error` is non-nil when `outFile.Write()` fails. Currently it will be `nil`, demonstrating the bug. Also assert that the returned byte count is 0 (not the partial count from the failed write).

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportEntrySwallowsWriteErrors"
**Test Language**: Go

### Demonstration

The test creates a file opened read-only (so all `Write` calls return "bad file descriptor" errors), then calls `ExportEntry` with a valid JSON-serializable struct. Despite both `outFile.Write(marshalled)` and `outFile.WriteString("\n")` failing with I/O errors, `ExportEntry` returns `nil` — proving that all 10+ export command callers will silently treat failed writes as successful row exports, potentially uploading truncated files to GCS.

### Test Body

```go
package cmd

import (
	"os"
	"testing"
)

// TestExportEntrySwallowsWriteErrors demonstrates that ExportEntry returns nil
// when the underlying file write fails, silently dropping rows from the export.
// A PASSING test means the bug IS present (error is swallowed).
func TestExportEntrySwallowsWriteErrors(t *testing.T) {
	// Create a temp file and open it read-only so Write() will fail.
	tmpFile, err := os.CreateTemp(t.TempDir(), "poc-write-err-*.jsonl")
	if err != nil {
		t.Fatalf("failed to create temp file: %v", err)
	}
	closedFileName := tmpFile.Name()
	tmpFile.Close()

	readOnlyFile, err := os.Open(closedFileName)
	if err != nil {
		t.Fatalf("failed to open file read-only: %v", err)
	}
	defer readOnlyFile.Close()

	// Verify the file truly rejects writes (precondition).
	_, writeErr := readOnlyFile.Write([]byte("test"))
	if writeErr == nil {
		t.Fatal("precondition failed: expected read-only file to reject writes")
	}
	t.Logf("Precondition confirmed: file rejects writes with: %v", writeErr)

	// A valid entry that will marshal to JSON without error.
	entry := map[string]string{"key": "value"}

	// Call ExportEntry with the read-only file — both Write calls will fail.
	_, exportErr := ExportEntry(entry, readOnlyFile, nil)

	// The bug: ExportEntry returns nil even though both writes failed.
	if exportErr != nil {
		t.Fatalf("bug appears fixed: ExportEntry returned error %v (expected nil due to swallowed error)", exportErr)
	}
	t.Log("BUG CONFIRMED: ExportEntry returned nil error despite file write failure — " +
		"callers will silently treat this as a successful row export")
}
```

### Test Output

```
=== RUN   TestExportEntrySwallowsWriteErrors
    data_integrity_poc_test.go:31: Precondition confirmed: file rejects writes with: write /var/folders/wz/c3l_zq0s6qscqln_qh5s00240000gn/T/TestExportEntrySwallowsWriteErrors2591524950/001/poc-write-err-3216846166.jsonl: bad file descriptor
time="2026-04-10T15:33:08.374-05:00" level=error msg="Error writing map[key:value] to file: write ...poc-write-err-3216846166.jsonl: bad file descriptor"
time="2026-04-10T15:33:08.374-05:00" level=error msg="Error writing new line to file ...poc-write-err-3216846166.jsonl: bad file descriptor"
    data_integrity_poc_test.go:43: BUG CONFIRMED: ExportEntry returned nil error despite file write failure — callers will silently treat this as a successful row export
--- PASS: TestExportEntrySwallowsWriteErrors (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	2.031s
```
