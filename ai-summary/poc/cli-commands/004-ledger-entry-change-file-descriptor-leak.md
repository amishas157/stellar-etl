# H004: Continuous ledger-entry-change exports eventually stop producing batches due to leaked file descriptors

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledger_entry_changes` should be able to run for long periods in continuous mode, closing each batch's output files after writing and uploading them so later ledger ranges continue exporting normally. The correct result is one JSON file per selected resource per batch, for the full runtime of the process.

## Mechanism

Two independent leaks compound here. `createOutputFile` creates missing files with `os.Create` and never closes that descriptor, and `exportTransformedData` opens each per-resource output file with `MustOutFile(path)` but never closes it before moving to the next resource or next batch. In continuous mode with upload enabled, `UploadTo` deletes each local file after upload, forcing the next batch to recreate and reopen fresh paths; over time, the process consumes the descriptor limit and later batches stop exporting entirely.

## Trigger

1. Run `export_ledger_entry_changes` without `--end-ledger` so it streams indefinitely, preferably with several export flags enabled.
2. Enable cloud upload so each batch deletes its local files after upload, causing the next batch to recreate them.
3. After enough batches, `os.Create` or `os.OpenFile` starts failing, and subsequent ledger ranges are missing from output because the command can no longer open batch files.

## Target Code

- `cmd/command_utils.go:createOutputFile:19-29` — calls `os.Create(filepath)` and discards the returned file without closing it
- `cmd/command_utils.go:MustOutFile:42-52` — always reopens the same path after `createOutputFile`
- `cmd/export_ledger_entry_changes.go:304-372` — opens `outFile := MustOutFile(path)` for each resource and never calls `outFile.Close()`
- `cmd/upload_to_gcs.go:71` — deletes local files after upload, ensuring the next batch recreates them

## Evidence

Unlike the one-shot exporters, `export_ledger_entry_changes` writes files inside a long-running per-batch loop and omits any `outFile.Close()` call in that loop. Combined with the leaked descriptor in `createOutputFile`, continuous exports recreate and reopen output paths over and over without releasing prior descriptors.

## Anti-Evidence

Short bounded exports may complete before hitting the process file limit, so the bug is easy to miss in normal tests. The loss appears only after enough batches accumulate, which fits the subsystem's continuous-streaming mode much more than its one-off batch commands.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `exportTransformedData` (export_ledger_entry_changes.go:295-377) and confirmed that `outFile` opened on line 311 via `MustOutFile(path)` is never closed. Compared all 10 export commands: all 9 siblings (`export_ledgers.go:63`, `export_transactions.go:56`, `export_operations.go:56`, `export_effects.go:59`, `export_trades.go:61`, `export_assets.go:72`, `export_contract_events.go:57`, `export_token_transfers.go:57`, `export_ledger_transaction.go:51`) call `outFile.Close()` after writing. `export_ledger_entry_changes` is the sole outlier. Additionally confirmed that `createOutputFile` (command_utils.go:22) discards the `*os.File` from `os.Create` without closing it.

### Code Paths Examined

- `cmd/command_utils.go:createOutputFile:19-29` — `os.Create(filepath)` return value assigned to `_`, fd leaked whenever file doesn't exist
- `cmd/command_utils.go:MustOutFile:31-53` — calls `createOutputFile` (leaks 1 fd) then `os.OpenFile` (returns 2nd fd to caller)
- `cmd/export_ledger_entry_changes.go:304-377` (`exportTransformedData`) — `outFile := MustOutFile(path)` on line 311, used for writes in inner loop, never closed; `MaybeUpload` on line 368 may delete the file while the fd is still open
- `cmd/export_ledger_entry_changes.go:79-291` (outer batch loop) — calls `exportTransformedData` for each batch indefinitely in continuous mode
- `cmd/export_ledgers.go:37,63` — reference pattern showing correct `outFile.Close()` call
- `cmd/upload_to_gcs.go:32,71` — `os.Open(path)` for upload also never closed (secondary leak), then `deleteLocalFiles` removes path

### Findings

**Bug 1 — `createOutputFile` fd leak (command_utils.go:22)**: `os.Create(filepath)` returns a `*os.File` that is discarded to `_`. The file descriptor is never closed. Since batch filenames embed start-end ledger numbers (`exportFilename` returns `%d-%d-%s.txt`), every batch creates unique paths, so this branch is taken every call — leaking 1 fd per resource per batch.

**Bug 2 — Missing `outFile.Close()` in `exportTransformedData` (export_ledger_entry_changes.go:311)**: This is a classic Pattern 3 outlier. All 9 sibling export commands call `outFile.Close()` after writing. `exportTransformedData` does not. Each iteration of the `for resource, output := range` loop opens a new file handle that is never explicitly closed — leaking 1 fd per resource per batch.

**Combined impact**: With N resource types enabled (up to 10), each batch leaks 2N file descriptors. At 20 fds/batch and a typical ulimit of 1024, exhaustion occurs after ~50 batches. When exhausted, `MustOutFile` calls `cmdLogger.Fatal` causing a crash (not silent loss).

**Correction to hypothesis**: The upload/delete cycle is not necessary to trigger the leak. Batch filenames are always unique (different ledger ranges), so `createOutputFile` always hits the `os.IsNotExist` branch regardless of upload. Also, exhaustion causes a Fatal crash rather than silent data loss. Go's GC finalizers may reclaim some fds nondeterministically, making the failure intermittent rather than guaranteed.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` (or create a new test file)
- **Setup**: Call `exportTransformedData` in a loop with mock data, using a small `ulimit -n` or counting open fds via `/proc/self/fd` (Linux) or `lsof` (macOS)
- **Steps**: (1) Create a transformedOutput map with several resource types, each containing a small slice of mock data. (2) Call `exportTransformedData` in a loop for 100+ iterations with unique start/end values. (3) After each iteration, count open file descriptors for the process.
- **Assertion**: Assert that open fd count grows linearly with iterations (confirming the leak). Alternatively, assert that after adding `outFile.Close()` to `exportTransformedData` and fixing `createOutputFile` to close its handle, the fd count remains stable across iterations.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportTransformedDataLeaksFileDescriptors"
**Test Language**: Go

### Demonstration

The test calls `exportTransformedData` 50 times with 3 resource types (accounts, offers, trustlines) and empty data slices, with GC disabled to prevent finalizer-based fd reclamation. After 50 iterations, the process has leaked exactly 300 file descriptors (2 per resource per iteration: 1 from `createOutputFile`'s discarded `os.Create` return and 1 from `MustOutFile`'s `os.OpenFile` that `exportTransformedData` never closes). This confirms both bugs and proves that continuous-mode exports with multiple resource types will exhaust the process fd limit after approximately `ulimit / (2 × resource_count)` batches.

### Test Body

```go
package cmd

import (
	"fmt"
	"os"
	"path/filepath"
	"runtime"
	"runtime/debug"
	"testing"
)

// countOpenFDs counts the number of open file descriptors for this process.
// Works on macOS (/dev/fd) and Linux (/proc/self/fd).
func countOpenFDs(t *testing.T) int {
	t.Helper()
	// Try /dev/fd first (macOS and some Linux), then /proc/self/fd (Linux)
	for _, dir := range []string{"/dev/fd", "/proc/self/fd"} {
		entries, err := os.ReadDir(dir)
		if err != nil {
			t.Logf("ReadDir(%s) failed: %v", dir, err)
			continue
		}
		return len(entries)
	}
	// Fallback: probe individual fd numbers
	count := 0
	for i := 0; i < 4096; i++ {
		_, err := os.Stat(filepath.Join("/dev/fd", fmt.Sprintf("%d", i)))
		if err == nil {
			count++
		} else {
			// fds are typically contiguous from 0; once we hit a gap after finding some, stop
			if count > 0 && i > count+10 {
				break
			}
		}
	}
	if count > 0 {
		return count
	}
	t.Fatal("cannot count open file descriptors on this platform")
	return 0
}

// TestExportTransformedDataLeaksFileDescriptors demonstrates that
// exportTransformedData leaks file descriptors because:
//  1. createOutputFile calls os.Create and discards the *os.File without closing
//  2. exportTransformedData never calls outFile.Close() on the file opened by MustOutFile
//
// Over many batch iterations (as in continuous mode), this exhausts the fd limit.
func TestExportTransformedDataLeaksFileDescriptors(t *testing.T) {
	// Disable GC so finalizers don't close leaked file descriptors
	originalGCPercent := debug.SetGCPercent(-1)
	defer debug.SetGCPercent(originalGCPercent)

	tmpDir := t.TempDir()
	outputFolder := filepath.Join(tmpDir, "output")
	parquetFolder := filepath.Join(tmpDir, "parquet")
	if err := os.MkdirAll(outputFolder, 0755); err != nil {
		t.Fatal(err)
	}
	if err := os.MkdirAll(parquetFolder, 0755); err != nil {
		t.Fatal(err)
	}

	// Force GC + finalizers before counting initial FDs to get a stable baseline
	runtime.GC()
	runtime.GC()

	initialFDs := countOpenFDs(t)

	const iterations = 50
	resourceTypes := []string{"accounts", "offers", "trustlines"}

	for i := 0; i < iterations; i++ {
		start := uint32(i * 100)
		end := uint32((i+1)*100 - 1)

		// Empty slices per resource — we just need the file open/close behavior
		transformedOutput := map[string][]interface{}{}
		for _, r := range resourceTypes {
			transformedOutput[r] = []interface{}{}
		}

		err := exportTransformedData(
			start, end,
			outputFolder, parquetFolder,
			transformedOutput,
			"", "", "", // no cloud credentials/bucket/provider
			nil,        // no extra fields
			false,      // no parquet
		)
		if err != nil {
			t.Fatalf("iteration %d: %v", i, err)
		}
	}

	finalFDs := countOpenFDs(t)
	leaked := finalFDs - initialFDs

	t.Logf("Initial FDs: %d, Final FDs: %d, Leaked: %d", initialFDs, finalFDs, leaked)
	t.Logf("Expected minimum leak: %d resource types × %d iterations × 2 fds (createOutputFile + MustOutFile) = %d",
		len(resourceTypes), iterations, len(resourceTypes)*iterations*2)

	// Each iteration opens 2 fds per resource type:
	//   1 from createOutputFile (os.Create, discarded to _)
	//   1 from MustOutFile (os.OpenFile, never closed by exportTransformedData)
	// With 3 resources and 50 iterations, expect at least 150 leaked fds.
	// We use a conservative threshold of iterations * len(resourceTypes) (150).
	minExpectedLeak := iterations * len(resourceTypes)
	if leaked < minExpectedLeak {
		t.Errorf("Expected at least %d leaked FDs (proving fd leak), but only found %d",
			minExpectedLeak, leaked)
	} else {
		t.Logf("CONFIRMED: %d file descriptors leaked across %d iterations — "+
			"exportTransformedData does not close outFile and createOutputFile discards os.Create fd",
			leaked, iterations)
	}
}
```

### Test Output

```
=== RUN   TestExportTransformedDataLeaksFileDescriptors
    data_integrity_poc_test.go:70: ReadDir(/dev/fd) failed: lstat /dev/fd/3: bad file descriptor
    data_integrity_poc_test.go:70: ReadDir(/proc/self/fd) failed: open /proc/self/fd: no such file or directory
    data_integrity_poc_test.go:98: ReadDir(/dev/fd) failed: lstat /dev/fd/3: bad file descriptor
    data_integrity_poc_test.go:98: ReadDir(/proc/self/fd) failed: open /proc/self/fd: no such file or directory
    data_integrity_poc_test.go:101: Initial FDs: 3, Final FDs: 303, Leaked: 300
    data_integrity_poc_test.go:102: Expected minimum leak: 3 resource types × 50 iterations × 2 fds (createOutputFile + MustOutFile) = 300
    data_integrity_poc_test.go:115: CONFIRMED: 300 file descriptors leaked across 50 iterations — exportTransformedData does not close outFile and createOutputFile discards os.Create fd
--- PASS: TestExportTransformedDataLeaksFileDescriptors (0.02s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	1.933s
```
