# 002: ExportEntry Swallows Write Errors

**Date**: 2026-04-10
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`ExportEntry` logs write failures from `outFile.Write` and `outFile.WriteString` but still returns `nil`. Every CLI exporter treats that `nil` as success, so a real filesystem failure can leave JSONL output truncated or malformed while success counters and later upload steps proceed as if the row was exported.

## Root Cause

`cmd.ExportEntry` has four fallible stages, but only the final `json.Marshal(i)` error is propagated. The earlier marshal/decode failures are logged and the write-path failures are also only logged, after which the function unconditionally returns `numBytes + newLineNumBytes, nil`.

## Reproduction

Run any JSON export command and hit a real write failure after the output file has been opened, such as `ENOSPC`, `EIO`, or another kernel-level `write(2)` failure. The exporter logs the write error, does not increment its failure counter, and can continue into later rows or upload logic with a partial JSONL artifact.

## Affected Code

- `cmd/command_utils.go:ExportEntry:55-86` — logs write failures and still returns `nil`
- `cmd/export_transactions.go:Run:43-49` — counts `ExportEntry` nil as a successful row
- `cmd/export_operations.go:Run:43-49` — same success-accounting pattern
- `cmd/export_effects.go:Run:45-51` — same success-accounting pattern
- `cmd/export_ledger_entry_changes.go:exportTransformedData:315-319` — same pattern before later processing/upload

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportEntrySwallowsWriteErrors`
- **Test language**: `go`
- **How to run**: `go build ./... && go test ./cmd/... -run '^TestExportEntrySwallowsWriteErrors$' -v`

### Test Body

```go
package cmd

import (
	"os"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

func TestExportEntrySwallowsWriteErrors(t *testing.T) {
	entry := transform.LedgerOutput{Sequence: 100}

	t.Run("pipe read end is not writable", func(t *testing.T) {
		r, w, err := os.Pipe()
		if err != nil {
			t.Fatalf("os.Pipe() failed: %v", err)
		}
		defer r.Close()
		if err := w.Close(); err != nil {
			t.Fatalf("closing pipe writer failed: %v", err)
		}

		numBytes, exportErr := ExportEntry(entry, r, nil)
		if exportErr != nil {
			t.Fatalf("expected swallowed write error, got: %v", exportErr)
		}
		if numBytes != 0 {
			t.Fatalf("expected zero bytes from failed writes, got %d", numBytes)
		}
	})

	t.Run("closed file reports success", func(t *testing.T) {
		tmpFile, err := os.CreateTemp("", "export-entry-poc-*")
		if err != nil {
			t.Fatalf("CreateTemp failed: %v", err)
		}
		tmpName := tmpFile.Name()
		defer os.Remove(tmpName)
		if err := tmpFile.Close(); err != nil {
			t.Fatalf("closing temp file failed: %v", err)
		}

		numBytes, exportErr := ExportEntry(entry, tmpFile, nil)
		if exportErr != nil {
			t.Fatalf("expected swallowed write error on closed file, got: %v", exportErr)
		}
		if numBytes != 0 {
			t.Fatalf("expected zero bytes from failed closed-file writes, got %d", numBytes)
		}
	})
}
```

## Expected vs Actual Behavior

- **Expected**: A failed JSON write should return a non-nil error so the caller can count the row as failed and stop or surface the export failure.
- **Actual**: `ExportEntry` logs the write failure but returns `nil`, so callers treat the row as successfully exported even though zero bytes were written.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `ExportEntry` path with real `*os.File` values whose writes fail inside the kernel and observes that the returned error is still `nil`.
2. Realistic preconditions: YES — the PoC uses deterministic stand-ins for the same class of write failures that can happen in normal exports (`ENOSPC`, `EIO`, closed/broken file descriptors).
3. Bug vs by-design: BUG — the function advertises an `error` return and every caller branches on that value for failure accounting, but the write path only logs via `Errorf`; even `--strict-export` cannot rescue this because `ExportEntry` bypasses `LogError`.
4. Final severity: Medium — the issue causes silent partial exports, malformed JSONL when newline writes fail, incorrect success metrics, and possible upload of truncated artifacts.
5. In scope: YES — it affects correctness of supported CLI export workflows.
6. Test correctness: CORRECT — the test uses production code and real OS file semantics, not mocks or circular assertions.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Return the first marshal, decode, write, or newline error from `ExportEntry` instead of logging and continuing. Callers already treat a non-nil return as export failure, so propagating the error is the minimal safe fix.
