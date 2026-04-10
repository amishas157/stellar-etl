# 007: JSON exporters ignore close failures

**Date**: 2026-04-10
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

All nine one-shot JSON export commands call `outFile.Close()` as a bare statement and discard its error. If the final file close surfaces a delayed writeback failure, the command still logs success stats and can continue into `MaybeUpload(...)`, so operators get a success-shaped completion after a failed file finalization.

## Root Cause

Each command writes JSON rows to a writable `*os.File`, then invokes `outFile.Close()` without checking the returned error. On writable files, `Close()` is part of the I/O error surface: filesystems can report deferred writeback failures only at close time, but these commands treat that path as success.

## Reproduction

Force a non-nil `Close()` error on the same `*os.File` type used by the exporters, then inspect the real command sources. The PoC demonstrates that `Close()` can fail after `ExportEntry(...)` writes succeed, and that every affected command still contains a bare `outFile.Close()` followed by the success/upload path.

## Affected Code

- `cmd/export_ledgers.go:ledgersCmd.Run:63-68` — ignores `outFile.Close()` and continues to stats/upload
- `cmd/export_transactions.go:transactionsCmd.Run:56-61` — same unchecked close pattern
- `cmd/export_operations.go:operationsCmd.Run:56-61` — same unchecked close pattern
- `cmd/export_effects.go:effectsCmd.Run:59-64` — same unchecked close pattern
- `cmd/export_trades.go:tradesCmd.Run:61-66` — same unchecked close pattern
- `cmd/export_assets.go:assetsCmd.Run:72-77` — same unchecked close pattern
- `cmd/export_contract_events.go:contractEventsCmd.Run:57-61` — same unchecked close pattern
- `cmd/export_token_transfers.go:tokenTransfersCmd.Run:57-62` — same unchecked close pattern
- `cmd/export_ledger_transaction.go:ledgerTransactionsCmd.Run:51-56` — same unchecked close pattern

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportCommandsIgnoreCloseError`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"runtime"
	"strings"
	"syscall"
	"testing"
)

func TestExportCommandsIgnoreCloseError(t *testing.T) {
	tmpFile, err := os.CreateTemp(t.TempDir(), "poc-close-error-*.jsonl")
	if err != nil {
		t.Fatal(err)
	}

	if _, err := ExportEntry(map[string]string{"ledger_id": "12345"}, tmpFile, nil); err != nil {
		t.Fatalf("ExportEntry failed: %v", err)
	}

	fd := tmpFile.Fd()
	if err := syscall.Close(int(fd)); err != nil {
		t.Fatalf("syscall.Close(fd=%d) failed: %v", fd, err)
	}

	closeErr := tmpFile.Close()
	if closeErr == nil {
		t.Fatal("expected Close() to return an error")
	}
	t.Logf("Close() returned error: %v", closeErr)

	_, currentFile, _, ok := runtime.Caller(0)
	if !ok {
		t.Fatal("could not determine current file path")
	}
	cmdDir := filepath.Dir(currentFile)

	affectedFiles := []string{
		"export_ledgers.go",
		"export_transactions.go",
		"export_operations.go",
		"export_effects.go",
		"export_trades.go",
		"export_assets.go",
		"export_contract_events.go",
		"export_token_transfers.go",
		"export_ledger_transaction.go",
	}

	for _, path := range affectedFiles {
		source, err := os.ReadFile(filepath.Join(cmdDir, path))
		if err != nil {
			t.Fatalf("reading %s: %v", path, err)
		}

		lines := strings.Split(string(source), "\n")
		closeLine := -1
		maybeUploadLine := -1
		for i, line := range lines {
			trimmed := strings.TrimSpace(line)
			if trimmed == "outFile.Close()" {
				closeLine = i
			}
			if closeLine != -1 && strings.Contains(trimmed, "MaybeUpload(") {
				maybeUploadLine = i
				break
			}
		}

		if closeLine == -1 {
			t.Fatalf("%s does not contain a bare outFile.Close() call", path)
		}
		if maybeUploadLine == -1 {
			t.Fatalf("%s does not continue to MaybeUpload() after outFile.Close()", path)
		}

		t.Logf("%s ignores outFile.Close() on line %d and still reaches MaybeUpload() on line %d", path, closeLine+1, maybeUploadLine+1)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: If `outFile.Close()` fails, the export command should return or fatal immediately and skip success stats and upload.
- **Actual**: The commands discard the `Close()` error, report success, and keep executing the post-close path.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC forces a real non-nil `Close()` result on a writable `*os.File`, and the traced production commands unconditionally ignore that result and proceed to `MaybeUpload(...)`.
2. Realistic preconditions: YES — the PoC uses `EBADF` only as a deterministic stand-in; real writable-file close failures can surface from delayed writeback on NFS, FUSE, or local filesystem error paths.
3. Bug vs by-design: BUG — no code or documentation indicates writable close failures are intentionally ignored, and doing so contradicts normal file-output error handling.
4. Final severity: Medium — this is an operational correctness/data-loss masking issue: the exporter can claim success after file finalization failed, but it does not directly remap financial fields.
5. In scope: YES — it is a concrete export-pipeline error-propagation flaw that can hide failed output generation.
6. Test correctness: CORRECT — the test does not assume the bug; it separately proves `Close()` can fail on the relevant object type and then checks the real command sources for the unchecked close-plus-continue path.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Replace every bare `outFile.Close()` in the one-shot JSON exporters with checked error handling, e.g. `if err := outFile.Close(); err != nil { cmdLogger.Fatal(...) }`, and only print success stats or call `MaybeUpload(...)` after a successful close.
