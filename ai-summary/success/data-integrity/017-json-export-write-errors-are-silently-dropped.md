# 017: JSON export write errors are silently dropped

**Date**: 2026-04-12
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`cmd.ExportEntry()` logs JSON write failures but still returns a `nil` error to its callers. The export commands use that return value to decide whether to increment failure counters, so a dropped row is counted as successful and later upload/summary steps still proceed.

I independently reproduced the primary `Write` failure path with a real `*os.File` pipe that returns `broken pipe`. The sibling newline path is also vulnerable by direct source trace: `WriteString("\n")` errors are logged and discarded in the same unconditional `return ..., nil` branch.

## Root Cause

`ExportEntry()` handles `json.Marshal`, `decoder.Decode`, `outFile.Write`, and `outFile.WriteString` errors inconsistently. Only the second `json.Marshal` failure is returned; the other error sites are logged with `cmdLogger.Errorf` and execution continues to `return numBytes + newLineNumBytes, nil`.

The export commands then gate `numFailures` updates on `if err != nil`, so a swallowed file-write error is indistinguishable from a successful row to the surrounding pipeline.

## Reproduction

During normal operation, any JSON export command that routes rows through `ExportEntry()` can hit this path when the output writer fails mid-run, such as a broken pipe, disk-full condition, or other file I/O error. The write failure is logged, but the caller receives `nil`, leaves its failure count unchanged, and can continue toward upload with a truncated JSONL artifact.

## Affected Code

- `cmd/command_utils.go:55-86` — `ExportEntry()` logs `Write`/`WriteString` failures and still returns `nil`
- `cmd/export_transactions.go:30-49` — transaction export only increments `numFailures` when `ExportEntry()` returns non-nil
- `cmd/export_ledgers.go:37-56` — ledger export uses the same success/failure accounting pattern
- `cmd/get_ledger_range_from_times.go:74-78` — standalone JSON output path ignores direct file write errors entirely

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportEntrySwallowsWriteError`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"os"
	"testing"
)

func TestExportEntrySwallowsWriteError(t *testing.T) {
	r, w, err := os.Pipe()
	if err != nil {
		t.Fatalf("os.Pipe() failed: %v", err)
	}
	r.Close()

	entry := map[string]string{"key": "value"}
	numBytes, exportErr := ExportEntry(entry, w, nil)

	if exportErr == nil {
		t.Fatalf("ExportEntry returned nil error despite write failure (numBytes=%d)", numBytes)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `ExportEntry()` should return the underlying file-write error so the caller can count the row as failed and stop or abort upload decisions.
- **Actual**: `ExportEntry()` logs the broken-pipe write failure and returns `nil`, so callers treat the dropped row as a successful export.

## Adversarial Review

1. Exercises claimed bug: YES — the test forces the real `outFile.Write` path in `ExportEntry()` to fail and shows the function still reports success to the caller
2. Realistic preconditions: YES — broken pipes, disk-full conditions, and other write-side file errors are ordinary operational failures for export jobs
3. Bug vs by-design: BUG — the function signature advertises `error`, callers rely on that contract for failure accounting, and no code or docs describe best-effort row writes as intentional
4. Final severity: Medium — the issue silently drops or truncates output rows under I/O failure, but it does not corrupt monetary values on successful writes
5. In scope: YES — this is a concrete silent-output-correctness failure in the export pipeline
6. Test correctness: CORRECT — it uses a real `*os.File` from `os.Pipe()`, the production `ExportEntry()` code path, and a failing assertion tied directly to the returned error contract
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Return the first non-nil error from `json.Marshal`, `decoder.Decode`, `outFile.Write`, and `outFile.WriteString` instead of only logging it. Callers should also treat `Close()` failures as fatal/export failures before reporting success or uploading the output file.
