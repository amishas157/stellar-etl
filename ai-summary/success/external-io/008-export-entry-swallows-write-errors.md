# 008: `ExportEntry()` swallows JSON write failures

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`ExportEntry()` logs `*os.File` write failures but still returns `nil`, so every JSON exporter that relies on its `(int, error)` result treats dropped or truncated rows as successful exports. When the output file stops accepting writes, failure counts stay too low and the CLI can continue on to upload a malformed or incomplete JSONL file.

## Root Cause

`cmd.ExportEntry()` propagates JSON marshaling failures from its second `json.Marshal`, but the actual file I/O errors from `outFile.Write()` and `outFile.WriteString("\n")` are only logged. The function then unconditionally returns `numBytes + newLineNumBytes, nil`, and representative callers in `export_ledgers`, `export_transactions`, and `export_ledger_entry_changes` only increment failure counts or abort when that error is non-nil.

## Reproduction

During normal operation, any real filesystem write failure while exporting JSON rows (for example, `ENOSPC` on a full disk or another write-side I/O failure) reaches the same `outFile.Write` / `outFile.WriteString` path. Because `ExportEntry()` swallows those errors, the row is treated as successfully exported even though the file can be empty, truncated, or missing its newline delimiter.

## Affected Code

- `cmd/command_utils.go:ExportEntry:55-86` — logs write failures and still returns `nil`
- `cmd/export_ledgers.go:Run:50-56` — increments `numFailures` only when `ExportEntry()` returns an error
- `cmd/export_transactions.go:Run:43-49` — same success/failure accounting pattern
- `cmd/export_ledger_entry_changes.go:exportTransformedData:316-319,368` — keeps processing and can still call `MaybeUpload()` after swallowed JSON write failures

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportEntrySwallowsWriteError`
- **Test language**: `go`
- **How to run**: Create `cmd/data_integrity_poc_test.go` with the body below, then run `go build ./... && go test ./cmd/... -run TestExportEntrySwallowsWriteError -v`.

### Test Body

```go
package cmd

import (
	"os"
	"testing"
)

func TestExportEntrySwallowsWriteError(t *testing.T) {
	tmpFile, err := os.CreateTemp(t.TempDir(), "export-entry-*.jsonl")
	if err != nil {
		t.Fatalf("create temp file: %v", err)
	}

	filePath := tmpFile.Name()
	if err := tmpFile.Close(); err != nil {
		t.Fatalf("close temp file: %v", err)
	}

	closedFile, err := os.OpenFile(filePath, os.O_RDWR, 0o644)
	if err != nil {
		t.Fatalf("reopen temp file: %v", err)
	}
	if err := closedFile.Close(); err != nil {
		t.Fatalf("close reopened file: %v", err)
	}

	numBytes, exportErr := ExportEntry(map[string]string{"key": "value"}, closedFile, nil)
	if exportErr != nil {
		t.Fatalf("expected swallowed write error, got %v", exportErr)
	}
	if numBytes != 0 {
		t.Fatalf("expected zero bytes reported on failed write, got %d", numBytes)
	}

	info, err := os.Stat(filePath)
	if err != nil {
		t.Fatalf("stat temp file: %v", err)
	}
	if info.Size() != 0 {
		t.Fatalf("expected empty file after failed write, got size %d", info.Size())
	}
}
```

## Expected vs Actual Behavior

- **Expected**: A failed JSON row write should return a non-nil error so the exporter can count the row as failed and avoid reporting a clean export.
- **Actual**: The underlying write fails, the output file remains empty, and `ExportEntry()` still returns `nil`, causing callers to treat the row as successfully exported.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `ExportEntry()` path with a real `*os.File` that returns OS-level write errors from both `Write` and `WriteString`.
2. Realistic preconditions: YES — the closed file in the test is only a deterministic stand-in for real filesystem write failures such as `ENOSPC` or other I/O errors; `ExportEntry()` handles all such write errors identically.
3. Bug vs by-design: BUG — the function signature and all representative callers expect file-write failures to be surfaced through the returned `error`, but only marshal failures are propagated.
4. Final severity: Medium — this causes masked export failure and possible malformed or incomplete JSON output, but it is an operational data-loss bug rather than a direct financial-value corruption bug.
5. In scope: YES — it is a concrete exporter correctness issue that can produce bad output while the command reports success.
6. Test correctness: CORRECT — the test uses the real production function, avoids circular assertions, and independently verifies that the file stayed empty after the failed write.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Return a wrapped error immediately when `outFile.Write()` or `outFile.WriteString("\n")` fails, and treat any short write as `io.ErrShortWrite`. The existing callers already have failure-accounting paths, so propagating the error will let them increment failure counts, stop when appropriate, and avoid uploading malformed JSON artifacts.
