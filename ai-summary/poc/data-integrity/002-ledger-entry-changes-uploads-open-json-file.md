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

---

## Final Review — Needs Revision

**Date**: 2026-04-11
**Final review by**: gpt-5.4, high

### What Needs Fixing

The current PoC proves only the structural fact that `exportTransformedData()` omits `outFile.Close()`. It does **not** prove the claimed corruption mechanism. `TestExportLedgerEntryChangesMissingFileClose` is a source-text grep, not a runtime reproduction, and `TestExportTransformedDataWritesWithoutClose` passes `cloudProvider=""`, so it never exercises `MaybeUpload()`'s upload path at all.

The runtime evidence also does not show stale or truncated output. `ExportEntry()` writes directly to `*os.File` with `Write`/`WriteString` rather than through a buffered writer, and a separate reader can observe the written bytes before `Close()`. In this review environment, a second reader saw the full file contents before close, which is a benign explanation for the observed behavior and undercuts the stronger "uploaded object is truncated/stale" claim.

### Revision Instructions

Either:

1. Reframe the finding narrowly as an **operational correctness** issue: `export_ledger_entry_changes` never closes JSON files, so close-time writeback errors are never observed and descriptors may remain open until GC/process exit; or
2. Keep the current framing, but replace the PoC with a **functional** demonstration that the uploaded/read-back bytes are wrong or that a close-time error is actually suppressed under realistic conditions.

For the next PoC attempt:

- Do not rely on source scanning alone.
- Exercise the real upload/read path, or otherwise show an explicit **expected vs actual** mismatch caused by the missing close.
- If you pursue the narrower operational-correctness framing, show a realistic observable consequence (for example, an unsurfaced close-time failure or measurable descriptor accumulation), not just the absence of a `Close()` call in source.

### Checks Passed So Far

- The code path is real: `exportTransformedData()` opens the JSON file with `MustOutFile()` and writes entries with `ExportEntry()` before calling `MaybeUpload()`, with no `outFile.Close()` in that function.
- The missing close is a structural outlier relative to sibling export commands, which do call `outFile.Close()` before upload.
- No documentation or inline comment reviewed during final review established that this ordering is intentional.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportTransformedDataLeaksFileDescriptors" and "TestExportTransformedDataFDsRecoveredOnlyByGC"
**Test Language**: Go

### Demonstration

Two functional tests demonstrate measurable file descriptor accumulation caused by the missing `Close()` in `exportTransformedData`. `TestExportTransformedDataLeaksFileDescriptors` calls the production function with 3 resource types, measures the process FD count before and after via `/dev/fd`, and proves 6 FDs leak (2 per resource — one from `createOutputFile` and one from `MustOutFile`, neither closed). `TestExportTransformedDataFDsRecoveredOnlyByGC` goes further: it shows the leaked FDs are only recovered when Go's garbage collector runs finalizers, proving the FD lifecycle is accidentally delegated to the GC rather than handled by explicit `Close()`. This means close-time I/O errors (NFS writeback, disk-full) are silently suppressed since finalizer-invoked `Close()` discards errors.

### Test Body

```go
package cmd

import (
	"os"
	"runtime"
	"runtime/debug"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// TestExportTransformedDataLeaksFileDescriptors demonstrates that
// exportTransformedData in export_ledger_entry_changes.go opens files via
// MustOutFile() but never calls outFile.Close(), causing file descriptor
// accumulation at the OS level. This is a functional test — not source
// scanning — that measures actual FD leak through the production code path.
//
// The missing Close() has three consequences:
//  1. File descriptors accumulate until Go's GC finalizer reclaims them
//  2. Close-time I/O errors (e.g. NFS writeback, disk-full) are silently lost
//  3. When MaybeUpload calls deleteLocalFiles, the FDs become zombies
//
// All 9 sibling export commands call outFile.Close() before MaybeUpload().
func TestExportTransformedDataLeaksFileDescriptors(t *testing.T) {
	// Disable automatic GC so os.File finalizers don't close leaked FDs
	// during our measurement window.
	oldGCPercent := debug.SetGCPercent(-1)
	defer debug.SetGCPercent(oldGCPercent)

	// Run GC twice to flush any pre-existing unreachable os.File objects
	// and their finalizers, establishing a clean FD baseline.
	runtime.GC()
	runtime.GC()

	tmpDir := t.TempDir()

	fdsBefore := countOpenFDs(t)

	// Call the production function with 3 resource types.
	// Each resource type opens one file via MustOutFile().
	// No cloud provider → MaybeUpload is a no-op (returns early).
	transformedOutput := map[string][]interface{}{
		"accounts": {
			transform.AccountOutput{AccountID: "GABC", Balance: 100.0},
		},
		"offers": {
			transform.OfferOutput{SellerID: "GABC", OfferID: 1},
		},
		"trustlines": {
			transform.TrustlineOutput{AccountID: "GABC"},
		},
	}
	numResources := len(transformedOutput)

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

	fdsAfter := countOpenFDs(t)
	leaked := fdsAfter - fdsBefore

	if leaked < numResources {
		t.Fatalf("Expected at least %d leaked FDs (one per resource type), but only %d extra FDs detected "+
			"(before=%d, after=%d). The bug may have been fixed — files are now properly closed.",
			numResources, leaked, fdsBefore, fdsAfter)
	}

	t.Logf("BUG CONFIRMED: %d file descriptors leaked after processing %d resource types (before=%d, after=%d)",
		leaked, numResources, fdsBefore, fdsAfter)
	t.Logf("Leaked FDs remain open until Go's GC finalizer runs, at which point close-time errors are silently discarded")
}

// TestExportTransformedDataFDsRecoveredOnlyByGC demonstrates that the FDs
// leaked by exportTransformedData are only recovered when Go's garbage
// collector runs finalizers — not by any explicit cleanup in the function.
// This proves the FD lifecycle is accidentally delegated to the GC.
func TestExportTransformedDataFDsRecoveredOnlyByGC(t *testing.T) {
	// Phase 1: Leak FDs (with GC disabled)
	oldGCPercent := debug.SetGCPercent(-1)
	runtime.GC()
	runtime.GC()

	tmpDir := t.TempDir()

	fdsBefore := countOpenFDs(t)

	transformedOutput := map[string][]interface{}{
		"accounts": {
			transform.AccountOutput{AccountID: "GABC", Balance: 100.0},
		},
		"offers": {
			transform.OfferOutput{SellerID: "GABC", OfferID: 1},
		},
		"trustlines": {
			transform.TrustlineOutput{AccountID: "GABC"},
		},
	}

	err := exportTransformedData(1, 10, tmpDir, tmpDir, transformedOutput, "", "", "", nil, false)
	if err != nil {
		t.Fatalf("exportTransformedData failed: %v", err)
	}

	fdsAfterCall := countOpenFDs(t)
	leakedBeforeGC := fdsAfterCall - fdsBefore

	if leakedBeforeGC < len(transformedOutput) {
		t.Fatalf("Precondition failed: expected FD leak but only %d extra FDs (before=%d, after=%d)",
			leakedBeforeGC, fdsBefore, fdsAfterCall)
	}

	t.Logf("Phase 1: %d FDs leaked (before=%d, after=%d)", leakedBeforeGC, fdsBefore, fdsAfterCall)

	// Phase 2: Re-enable GC and force collection. The os.File finalizers
	// should close the leaked FDs, reducing the count.
	debug.SetGCPercent(oldGCPercent)
	runtime.GC()
	runtime.GC()

	fdsAfterGC := countOpenFDs(t)
	recoveredByGC := fdsAfterCall - fdsAfterGC

	if recoveredByGC <= 0 {
		t.Fatalf("Expected GC to recover leaked FDs, but FD count did not decrease (afterCall=%d, afterGC=%d)",
			fdsAfterCall, fdsAfterGC)
	}

	t.Logf("Phase 2: GC recovered %d FDs (afterCall=%d, afterGC=%d)", recoveredByGC, fdsAfterCall, fdsAfterGC)
	t.Logf("BUG CONFIRMED: exportTransformedData relies on GC finalizers for file cleanup instead of explicit Close()")
}

// countOpenFDs returns the number of open file descriptors for the current process.
// Uses Readdirnames (not ReadDir/Lstat) to avoid races with transient FDs.
// Works on macOS (/dev/fd) and Linux (/dev/fd or /proc/self/fd).
func countOpenFDs(t *testing.T) int {
	t.Helper()
	f, err := os.Open("/dev/fd")
	if err != nil {
		t.Fatalf("cannot open /dev/fd: %v", err)
	}
	defer f.Close()
	names, err := f.Readdirnames(-1)
	if err != nil {
		t.Fatalf("cannot read /dev/fd entries: %v", err)
	}
	return len(names)
}
```

### Test Output

```
=== RUN   TestExportTransformedDataLeaksFileDescriptors
    data_integrity_poc_test.go:77: BUG CONFIRMED: 6 file descriptors leaked after processing 3 resource types (before=6, after=12)
    data_integrity_poc_test.go:79: Leaked FDs remain open until Go's GC finalizer runs, at which point close-time errors are silently discarded
--- PASS: TestExportTransformedDataLeaksFileDescriptors (0.00s)
=== RUN   TestExportTransformedDataFDsRecoveredOnlyByGC
    data_integrity_poc_test.go:121: Phase 1: 6 FDs leaked (before=6, after=12)
    data_integrity_poc_test.go:137: Phase 2: GC recovered 6 FDs (afterCall=12, afterGC=6)
    data_integrity_poc_test.go:138: BUG CONFIRMED: exportTransformedData relies on GC finalizers for file cleanup instead of explicit Close()
--- PASS: TestExportTransformedDataFDsRecoveredOnlyByGC (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	2.074s
```
