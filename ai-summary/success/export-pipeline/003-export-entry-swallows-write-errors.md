# 003: ExportEntry swallows JSON write errors

**Date**: 2026-04-10
**Severity**: Medium
**Impact**: Silent partial exports after I/O errors
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`ExportEntry()` logs failures from both `outFile.Write(...)` and `WriteString("\n")` but still returns `nil`. Every export command trusts that return value to decide whether a row exported successfully, so write-time I/O failures silently drop rows while the caller keeps counting success and can still upload the incomplete file.

## Root Cause

`ExportEntry()` only propagates errors from the second `json.Marshal(...)` call. The two filesystem writes at the end of the function overwrite `err`, log it, and then fall through to `return numBytes + newLineNumBytes, nil`, so the caller cannot distinguish a successful row export from a failed one.

## Reproduction

Any export command that hits a write-time filesystem failure on its JSON output path will mis-handle the row. A read-only file descriptor, `ENOSPC`, or similar I/O fault causes the row write to fail, but the exporter still treats the row as successful and continues toward its normal completion and upload path.

## Affected Code

- `cmd/command_utils.go:ExportEntry:55-86` — logs `Write` and `WriteString` errors but returns `nil`.
- `cmd/export_transactions.go:transactionsCmd.Run:43-49` — representative caller that increments failures only when `ExportEntry` returns a non-nil error.
- `cmd/export_ledger_entry_changes.go:exportTransformedData:316-319` — intended abort-on-export-error path never triggers for JSON write failures.
- `internal/utils/logger.go:EtlLogger.LogError:17-23` — strict export mode can only fatal on errors that reach the caller, which these do not.

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportEntrySwallowsWriteErrors`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `cmd/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./cmd/... -run TestExportEntrySwallowsWriteErrors -v`
  4. Observe that the output file remains empty even though `ExportEntry` returns a nil error.

### Test Body

```go
package cmd

import (
	"os"
	"testing"
)

func TestExportEntrySwallowsWriteErrors(t *testing.T) {
	tmpFile, err := os.CreateTemp(t.TempDir(), "poc-write-err-*.jsonl")
	if err != nil {
		t.Fatalf("failed to create temp file: %v", err)
	}

	fileName := tmpFile.Name()
	if err := tmpFile.Close(); err != nil {
		t.Fatalf("failed to close temp file: %v", err)
	}

	readOnlyFile, err := os.Open(fileName)
	if err != nil {
		t.Fatalf("failed to reopen file read-only: %v", err)
	}
	defer readOnlyFile.Close()

	if _, err := readOnlyFile.Write([]byte("test")); err == nil {
		t.Fatal("precondition failed: expected read-only file to reject writes")
	}

	numBytes, exportErr := ExportEntry(map[string]string{"key": "value"}, readOnlyFile, nil)
	if exportErr != nil {
		t.Fatalf("bug appears fixed: ExportEntry returned %v", exportErr)
	}
	if numBytes != 0 {
		t.Fatalf("expected zero bytes reported after failed writes, got %d", numBytes)
	}

	written, err := os.ReadFile(fileName)
	if err != nil {
		t.Fatalf("failed to read temp file: %v", err)
	}
	if len(written) != 0 {
		t.Fatalf("expected file to remain empty after failed writes, got %q", written)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `ExportEntry` should return a non-nil error when either the JSON row write or trailing newline write fails, so the caller can fail the row or abort the export.
- **Actual**: `ExportEntry` returns `nil` even when both writes fail and no bytes reach disk, so callers treat the row as successfully exported.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC invokes the real `ExportEntry` implementation and forces both terminal filesystem writes to fail.
2. Realistic preconditions: YES — the reproducer uses a read-only file descriptor, but the same code path is what a real `ENOSPC` or output-path I/O error would hit during exports.
3. Bug vs by-design: BUG — the function signature returns `error`, callers branch on that error to count failures, and strict export mode depends on the error reaching the caller.
4. Final severity: Medium — this is silent export loss under specific I/O faults rather than a direct monetary field corruption.
5. In scope: YES — the bug produces wrong export outcomes inside this repository without relying on upstream SDK behavior.
6. Test correctness: CORRECT — it uses a real `*os.File` that independently proves writes fail, then confirms the production function still reports success and leaves the file empty.
7. Alternative explanations: NONE — the empty file is fully explained by the failed writes, and the nil return is independently caused by `ExportEntry` swallowing those errors.
8. Novelty: PREVIOUSLY REPORTED IN REPO HISTORY — the issue is still real; duplicate handling is external to this review step.

## Suggested Fix

Return write-time errors from `outFile.Write(...)` and `WriteString("\n")` instead of logging and continuing. If the JSON write fails, skip the newline write, return the persisted byte count plus a wrapped error, and let callers decide whether to abort or suppress upload of the incomplete artifact.
