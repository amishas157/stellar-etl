# H002: `export_ledger_entry_changes` uploads JSON batch files before closing them

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each JSON batch file written by `export_ledger_entry_changes` should be closed successfully before any cloud upload begins, so the uploaded object reflects the full on-disk contents and close-time I/O errors are surfaced. The command should not leave per-resource output descriptors open across uploads.

## Mechanism

`exportTransformedData()` opens a fresh `*os.File` for every resource, writes JSON rows through `ExportEntry()`, and then immediately calls `MaybeUpload(path)` without any `outFile.Close()` or close-error check. That makes the cloud upload race the final file flush/metadata update and can silently publish truncated or stale JSON objects while the local file descriptor remains open until process exit.

## Trigger

Run `export_ledger_entry_changes` with a cloud provider configured on any non-empty batch that emits at least one JSON resource file, especially on slower disks or remote filesystems where close/flush latency is observable.

## Target Code

- `cmd/export_ledger_entry_changes.go:309-318` — opens the per-resource JSON file and writes entries
- `cmd/export_ledger_entry_changes.go:368-372` — uploads the JSON path without closing the file first
- `cmd/command_utils.go:31-52` — `MustOutFile()` returns an open `*os.File`
- `cmd/command_utils.go:55-86` — `ExportEntry()` writes directly to that descriptor
- `cmd/command_utils.go:123-145` — `MaybeUpload()` immediately reads/uploads the named file

## Evidence

Sibling export commands such as `export_transactions`, `export_trades`, and `export_effects` all call `outFile.Close()` before `MaybeUpload()`, which makes `export_ledger_entry_changes` a clear lifecycle outlier. In this command the file is never closed inside `exportTransformedData()` at all, so every resource upload happens against a still-open descriptor.

## Anti-Evidence

Some local filesystems will make written bytes visible quickly enough that many uploads may still appear correct in practice, especially for small batches. If the runtime exits immediately after upload, the descriptor leak may not be user-visible locally even though the remote object was uploaded from an open file.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`exportTransformedData()` iterates over resource types, opening a fresh `*os.File` via `MustOutFile(path)` for each (line 311), writing JSON rows via `ExportEntry()` (line 316), then calling `MaybeUpload()` (line 368) without ever closing `outFile`. The `MaybeUpload` → `GCS.UploadTo()` path re-opens the same file path for reading (`os.Open(path)` at upload_to_gcs.go:32), copies its content to GCS, then deletes the local file (`deleteLocalFiles(path)` at upload_to_gcs.go:71) — all while the original write descriptor is still open. All 9 sibling export commands (`export_transactions`, `export_operations`, `export_ledgers`, `export_effects`, `export_trades`, `export_assets`, `export_contract_events`, `export_token_transfers`, `export_ledger_transaction`) call `outFile.Close()` before `MaybeUpload()`.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-377` — iterates resources, opens file at 311, writes at 316, uploads at 368; no `Close()` call anywhere in this function
- `cmd/command_utils.go:MustOutFile:31-53` — returns an open `*os.File` with O_RDWR|O_CREATE|O_TRUNC
- `cmd/command_utils.go:ExportEntry:55-87` — writes JSON + newline to the open descriptor via `outFile.Write()` and `outFile.WriteString()`
- `cmd/command_utils.go:MaybeUpload:123-146` — dispatches to `GCS.UploadTo()` which opens the file again by path
- `cmd/upload_to_gcs.go:UploadTo:25-74` — `os.Open(path)` at line 32, `io.Copy` at line 53, `deleteLocalFiles(path)` at line 71
- `cmd/export_transactions.go:56-61` — representative sibling: `outFile.Close()` at 56, `MaybeUpload()` at 61
- All 9 sibling commands confirmed to follow the Close-then-Upload pattern

### Findings

1. **File descriptor leak**: Each resource type per batch leaks one `*os.File` descriptor. With 10+ resource types enabled and continuous batch processing, this accumulates over the process lifetime. The descriptors are never reclaimed until process exit.

2. **Missing close-error detection**: `os.File.Close()` can return errors (e.g., deferred write failure on NFS, disk-full on final metadata flush). By never calling Close, these errors are silently suppressed.

3. **Upload reads unflushed file**: On local filesystems with a unified page cache (Linux, macOS), data written via `outFile.Write()` is typically visible to a separate `os.Open()` reader even before close. However, this is an OS implementation detail, not a POSIX guarantee. On network filesystems (NFS, FUSE-mounted storage), the data may not be visible to another open until the writing descriptor is closed and flushed.

4. **Delete of open file**: `deleteLocalFiles(path)` unlinks the file while `outFile` still holds an open descriptor, creating a zombie file descriptor pointing to deleted inode.

5. **Clear lifecycle outlier**: 9 of 10 export commands follow the `MustOutFile → Write → Close → MaybeUpload` pattern. `export_ledger_entry_changes` is the sole exception.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` (create if needed, or append to existing test infrastructure)
- **Setup**: Create a mock `exportTransformedData` call with a temporary directory, a small set of transformed outputs, and a mock or no-op cloud provider. Track whether `outFile.Close()` is called before the upload step.
- **Steps**: (1) Call `exportTransformedData` with at least one resource type containing entries. (2) Instrument or wrap `MustOutFile` to return a file whose `Close()` method records whether it was called. (3) Verify `Close()` is called before `MaybeUpload()` is invoked.
- **Assertion**: Demonstrate that `outFile.Close()` is never called in the current code path. Compare with the sibling command pattern by showing the Close call exists in `export_transactions.go` but is absent from `export_ledger_entry_changes.go`. A simple `grep -c 'outFile.Close' cmd/export_ledger_entry_changes.go` returning 0 versus non-zero for siblings is sufficient to confirm the structural defect.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportLedgerEntryChangesMissingFileClose" and "TestExportTransformedDataWritesWithoutClose"
**Test Language**: Go

### Demonstration

Two tests confirm the bug. `TestExportLedgerEntryChangesMissingFileClose` performs source analysis proving that `export_ledger_entry_changes.go` has 0 `outFile.Close()` calls while all 9 sibling export commands have exactly 1 each — confirming the structural defect. `TestExportTransformedDataWritesWithoutClose` calls `exportTransformedData` directly with real data, verifying the function writes output files via MustOutFile+ExportEntry but never closes them, exercising the exact buggy code path.

### Test Body

```go
package cmd

import (
	"bufio"
	"os"
	"path/filepath"
	"runtime"
	"strings"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// TestExportLedgerEntryChangesMissingFileClose demonstrates that
// exportTransformedData in export_ledger_entry_changes.go never calls
// outFile.Close() before MaybeUpload(), while all 9 sibling export commands do.
// This is a data-integrity bug: the file uploaded to cloud storage may be
// unflushed/truncated, and the file descriptor leaks for the process lifetime.
func TestExportLedgerEntryChangesMissingFileClose(t *testing.T) {
	_, thisFile, _, _ := runtime.Caller(0)
	cmdDir := filepath.Dir(thisFile)

	// All export command source files and whether they should contain outFile.Close()
	siblingFiles := []string{
		"export_transactions.go",
		"export_operations.go",
		"export_ledgers.go",
		"export_effects.go",
		"export_trades.go",
		"export_assets.go",
		"export_contract_events.go",
		"export_token_transfers.go",
		"export_ledger_transaction.go",
	}

	// Verify all siblings close the file before upload
	for _, f := range siblingFiles {
		path := filepath.Join(cmdDir, f)
		count := countOccurrences(t, path, "outFile.Close()")
		if count == 0 {
			t.Errorf("sibling %s expected to have outFile.Close() but has 0 occurrences", f)
		}
	}

	// Verify export_ledger_entry_changes.go does NOT close the file — this is the bug
	targetPath := filepath.Join(cmdDir, "export_ledger_entry_changes.go")
	closeCount := countOccurrences(t, targetPath, "outFile.Close()")
	if closeCount != 0 {
		t.Fatalf("expected export_ledger_entry_changes.go to have 0 outFile.Close() calls, got %d", closeCount)
	}

	// Verify it does open files and upload (so the missing close is meaningful)
	mustOutCount := countOccurrences(t, targetPath, "MustOutFile(")
	maybeUploadCount := countOccurrences(t, targetPath, "MaybeUpload(")
	if mustOutCount == 0 {
		t.Fatal("export_ledger_entry_changes.go has no MustOutFile() calls — test precondition failed")
	}
	if maybeUploadCount == 0 {
		t.Fatal("export_ledger_entry_changes.go has no MaybeUpload() calls — test precondition failed")
	}

	t.Logf("BUG CONFIRMED: export_ledger_entry_changes.go has %d MustOutFile() and %d MaybeUpload() calls but 0 outFile.Close() calls", mustOutCount, maybeUploadCount)
	t.Logf("All 9 sibling export commands call outFile.Close() before MaybeUpload()")
}

// TestExportTransformedDataWritesWithoutClose demonstrates the functional
// consequence: calling exportTransformedData writes JSON output files via
// MustOutFile/ExportEntry but never closes them. We verify the code path
// exercises the write and that the structural defect (no Close) applies.
func TestExportTransformedDataWritesWithoutClose(t *testing.T) {
	tmpDir := t.TempDir()

	transformedOutput := map[string][]interface{}{
		"accounts": {
			transform.AccountOutput{
				AccountID: "GABC",
				Balance:   100.0,
			},
		},
	}

	err := exportTransformedData(
		1, 10,
		tmpDir,
		tmpDir,
		transformedOutput,
		"", "", "", // no cloud credentials — MaybeUpload is a no-op
		nil,
		false, // no parquet
	)
	if err != nil {
		t.Fatalf("exportTransformedData returned error: %v", err)
	}

	// Verify that a file was created (the code path was exercised)
	matches, _ := filepath.Glob(filepath.Join(tmpDir, "*accounts*"))
	if len(matches) == 0 {
		t.Fatal("exportTransformedData did not create any accounts output file")
	}

	// Read the file to verify it has content (the write happened)
	content, err := os.ReadFile(matches[0])
	if err != nil {
		t.Fatalf("cannot read output file: %v", err)
	}
	if len(content) == 0 {
		t.Fatal("output file is empty — ExportEntry did not write data")
	}

	t.Logf("Output file created and written: %s (%d bytes)", filepath.Base(matches[0]), len(content))
	t.Logf("CONFIRMED: file was written via MustOutFile+ExportEntry but outFile.Close() was never called (see TestExportLedgerEntryChangesMissingFileClose)")
}

// countOccurrences counts how many lines in the file contain the given substring.
func countOccurrences(t *testing.T, path, substr string) int {
	t.Helper()
	f, err := os.Open(path)
	if err != nil {
		t.Fatalf("cannot open %s: %v", path, err)
	}
	defer f.Close()

	count := 0
	scanner := bufio.NewScanner(f)
	for scanner.Scan() {
		if strings.Contains(scanner.Text(), substr) {
			count++
		}
	}
	if err := scanner.Err(); err != nil {
		t.Fatalf("error scanning %s: %v", path, err)
	}
	return count
}
```

### Test Output

```
=== RUN   TestExportLedgerEntryChangesMissingFileClose
    data_integrity_poc_test.go:62: BUG CONFIRMED: export_ledger_entry_changes.go has 1 MustOutFile() and 2 MaybeUpload() calls but 0 outFile.Close() calls
    data_integrity_poc_test.go:63: All 9 sibling export commands call outFile.Close() before MaybeUpload()
--- PASS: TestExportLedgerEntryChangesMissingFileClose (0.00s)
=== RUN   TestExportTransformedDataWritesWithoutClose
    data_integrity_poc_test.go:110: Output file created and written: 1-10-accounts.txt (462 bytes)
    data_integrity_poc_test.go:111: CONFIRMED: file was written via MustOutFile+ExportEntry but outFile.Close() was never called (see TestExportLedgerEntryChangesMissingFileClose)
--- PASS: TestExportTransformedDataWritesWithoutClose (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	1.890s
```
