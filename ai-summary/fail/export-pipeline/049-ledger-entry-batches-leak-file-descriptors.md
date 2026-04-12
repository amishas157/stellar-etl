# H003: `export_ledger_entry_changes` can lose later batches after leaking file descriptors

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each per-resource batch file opened by `export_ledger_entry_changes` should be closed once the batch has been written and optionally uploaded. Long-running exports over many batches should therefore be able to emit every expected file instead of failing partway through with open-file exhaustion.

## Mechanism

`MustOutFile()` first calls `createOutputFile()`, which uses `os.Create()` without closing the returned descriptor, then opens the same path again with `os.OpenFile()`. `exportTransformedData()` never closes that second `outFile` at all, so every resource file in every batch leaks at least one live descriptor and usually two. On sufficiently long exports, later file opens fail with `EMFILE`, which aborts remaining resources/batches and leaves a partial export on disk or in cloud storage.

## Trigger

Run `export_ledger_entry_changes` over a long range with a small `--batch-size` and multiple `--export-*` resource types enabled. As the command iterates batches, it opens a new JSON file for each resource and never closes it; once the process crosses the OS open-file limit, later `MustOutFile()` or parquet opens will fail and the remaining batch artifacts will be missing.

## Target Code

- `cmd/command_utils.go:19-25` — `createOutputFile()` calls `os.Create()` and drops the returned file handle
- `cmd/command_utils.go:31-52` — `MustOutFile()` opens the same path again and returns a second handle
- `cmd/export_ledger_entry_changes.go:309-372` — `exportTransformedData()` writes, uploads, and returns without ever calling `outFile.Close()`

## Evidence

The leak is visible directly in the control flow: after `outFile := MustOutFile(path)`, the function writes rows, optionally uploads JSON and Parquet, and exits the loop body without any close call. This command is uniquely susceptible because it opens one file per resource per batch, so descriptor growth scales with both range length and number of exported entity types rather than staying at a single file per process.

## Anti-Evidence

Short runs may never accumulate enough open files to hit the OS limit, and process exit will eventually release the descriptors. But that only masks the issue on small workloads; the bug still turns perfectly valid large exports into partial datasets once the descriptor ceiling is reached.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — prior fail/005 investigated a different hypothesis (upload-before-close → partial GCS uploads) and was rejected for the wrong mechanism; its lesson learned explicitly called out FD leaks as the real issue and suggested refiling. This hypothesis correctly targets the FD leak itself.

### Trace Summary

`exportTransformedData()` iterates over each resource type in `transformedOutput`, calling `MustOutFile(path)` on line 311 which returns an `*os.File`. The function uses this handle for `ExportEntry()` writes, then calls `MaybeUpload()` and optionally `WriteParquet()`, but never calls `outFile.Close()`. Meanwhile, `MustOutFile()` internally calls `createOutputFile()` which leaks an additional FD via `os.Create()` with a discarded return value. Every other export command in the `cmd/` package calls `outFile.Close()` after writing — this command is the sole outlier. Additionally, `UploadTo()` opens a third FD via `os.Open(path)` for reading during upload and also never closes it.

### Code Paths Examined

- `cmd/command_utils.go:createOutputFile:19-29` — `os.Create(filepath)` return assigned to `_`, leaking one FD per new file
- `cmd/command_utils.go:MustOutFile:31-52` — calls `createOutputFile` then `os.OpenFile`, returning second handle; caller must close
- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-377` — loops over resource types, opens file at line 311, writes entries, uploads, writes parquet, but NO `outFile.Close()` anywhere in the loop body or after it
- `cmd/upload_to_gcs.go:UploadTo:25-74` — opens a third FD via `os.Open(path)` at line 32 without closing `reader`; calls `deleteLocalFiles(path)` which unlinks the file but does not release open FDs
- `cmd/export_ledgers.go:63`, `cmd/export_transactions.go:56`, `cmd/export_operations.go:56`, `cmd/export_effects.go:59`, `cmd/export_trades.go:61`, `cmd/export_assets.go:72`, `cmd/export_contract_events.go:57`, `cmd/export_token_transfers.go:57`, `cmd/export_ledger_transaction.go:51` — all nine one-shot export commands call `outFile.Close()`, confirming `export_ledger_entry_changes` is the sole outlier

### Findings

1. **Missing `outFile.Close()` in `exportTransformedData()`**: Confirmed. The function opens a file per resource type per batch via `MustOutFile()` and never closes it. With 10 resource types enabled and a batch size of 64, processing 10,000 ledgers (~156 batches) leaks ~1,560 FDs from `outFile` alone.

2. **`createOutputFile()` leaks an additional FD**: Confirmed. When the file doesn't exist (the common case since filenames encode batch ranges), `os.Create()` opens a new FD and the return value is discarded. This doubles the leak to ~3,120 FDs for the same scenario.

3. **`UploadTo()` leaks a third FD**: The `reader` opened via `os.Open(path)` on line 32 is never closed. When GCS upload is enabled, each resource file leaks 3 FDs total. `deleteLocalFiles(path)` unlinks the file from the filesystem but does not release open kernel FDs.

4. **Fatal exit on exhaustion**: When `MustOutFile()` fails (e.g., `os.OpenFile` returns EMFILE), it calls `cmdLogger.Fatal()` which terminates the process. All subsequent batches are lost, producing a partial export.

5. **Consistency with other commands**: All nine one-shot export commands call `outFile.Close()`. This is a Pattern 3 (export command consistency) deviation — the batch-processing command is the only one that omits it.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` (create if not present; alternatively `cmd/command_utils_test.go`)
- **Setup**: Call `exportTransformedData()` in a loop with dummy `transformedOutput` containing a single resource type, using a temporary directory for `folderPath`. Before the loop, record the process FD count via `/proc/self/fd` (Linux) or `lsof -p` (macOS).
- **Steps**: Run `exportTransformedData()` N times (e.g., 100) with distinct batch start/end values so each call creates a new file. After all iterations, count open FDs again.
- **Assertion**: Assert that the FD count after the loop exceeds the initial count by at least N (proving descriptors accumulate). For a stronger assertion, set `ulimit -n` to a low value (e.g., 128) and verify that the loop fatals before completing all iterations.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportTransformedDataLeaksFileDescriptors"
**Test Language**: Go

### Demonstration

The test calls `exportTransformedData()` 50 times with distinct batch ranges and a single resource type, counting open file descriptors before and after the loop. It observed 100 leaked FDs (2 per call: one from `createOutputFile()`'s discarded `os.Create()` and one from the unclosed `outFile` returned by `MustOutFile()`). This confirms that FDs accumulate linearly with batch count and are never released, proving the hypothesis that long-running exports will exhaust the OS file descriptor limit.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"runtime"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// countOpenFDs returns the number of open file descriptors for this process.
// On macOS /dev/fd is a virtual directory listing the current process's FDs.
// On Linux /proc/self/fd serves the same purpose.
func countOpenFDs(t *testing.T) int {
	t.Helper()
	var fdDir string
	switch runtime.GOOS {
	case "darwin":
		fdDir = "/dev/fd"
	case "linux":
		fdDir = "/proc/self/fd"
	default:
		t.Skipf("unsupported OS for FD counting: %s", runtime.GOOS)
	}

	dir, err := os.Open(fdDir)
	if err != nil {
		t.Fatalf("cannot open %s: %v", fdDir, err)
	}
	// Use Readdirnames to avoid lstat calls on volatile /dev/fd entries
	names, err := dir.Readdirnames(-1)
	dir.Close()
	if err != nil {
		t.Fatalf("cannot read %s: %v", fdDir, err)
	}
	return len(names)
}

// TestExportTransformedDataLeaksFileDescriptors demonstrates that
// exportTransformedData() never closes the outFile it opens via MustOutFile(),
// causing file descriptor leaks that accumulate over repeated calls.
// Each call also leaks an extra FD from createOutputFile()'s discarded os.Create().
func TestExportTransformedDataLeaksFileDescriptors(t *testing.T) {
	tmpDir := t.TempDir()
	folderPath := filepath.Join(tmpDir, "json")
	parquetFolderPath := filepath.Join(tmpDir, "parquet")

	// Build minimal transformedOutput with one resource type and one entry.
	// Using TtlOutput since it's a small struct.
	dummyEntry := transform.TtlOutput{
		KeyHash:            "abc123",
		LiveUntilLedgerSeq: 100,
		LastModifiedLedger: 50,
	}
	transformedOutput := map[string][]interface{}{
		"ttl": {dummyEntry},
	}

	const iterations = 50

	// Record baseline FD count before the loop.
	fdBefore := countOpenFDs(t)
	t.Logf("FDs before loop: %d", fdBefore)

	for i := 0; i < iterations; i++ {
		start := uint32(i * 64)
		end := start + 63
		err := exportTransformedData(
			start, end,
			folderPath,
			parquetFolderPath,
			transformedOutput,
			"", "", "", // no cloud upload
			nil,   // no extra fields
			false, // no parquet
		)
		if err != nil {
			t.Fatalf("exportTransformedData iteration %d failed: %v", i, err)
		}
	}

	fdAfter := countOpenFDs(t)
	t.Logf("FDs after %d iterations: %d", iterations, fdAfter)

	leaked := fdAfter - fdBefore
	t.Logf("Leaked FDs: %d (expected at least %d)", leaked, iterations)

	// Each iteration should leak at least 1 FD (the outFile that is never closed).
	// In practice it leaks 2 per iteration (createOutputFile's os.Create + MustOutFile's os.OpenFile).
	// We assert conservatively: at least 1 leaked FD per iteration.
	if leaked < iterations {
		t.Errorf("Expected at least %d leaked FDs, but only found %d. "+
			"The FD leak may have been fixed.", iterations, leaked)
	}
}
```

### Test Output

```
=== RUN   TestExportTransformedDataLeaksFileDescriptors
    data_integrity_poc_test.go:64: FDs before loop: 6
    data_integrity_poc_test.go:84: FDs after 50 iterations: 106
    data_integrity_poc_test.go:87: Leaked FDs: 100 (expected at least 50)
--- PASS: TestExportTransformedDataLeaksFileDescriptors (0.01s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.612s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-12
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Exercises claimed issue**: PASS — I appended the PoC test to `cmd/data_integrity_poc_test.go` and independently reproduced the same growth pattern twice (`leaked=100` after 50 iterations). The test drives the real `exportTransformedData()` helper on the production code path.
2. **Realistic preconditions**: PASS — `export_ledger_entry_changes` is designed to emit a distinct file per `(batch, resource)` pair, so repeated opens during long bounded runs or continuous mode are normal operation, not a contrived setup.
3. **Bug vs by design**: PASS — sibling one-shot exporters explicitly close their JSON writers, while `exportTransformedData()` does not. The Go runtime also documents that garbage collection may close descriptors only later via finalizers, which is not an intentional resource-management contract for this command.
4. **Impact / severity**: PASS — Medium is the right classification. This is an operational correctness issue that can halt later exports once the process reaches the FD limit; it is not direct field-value corruption.
5. **In scope**: PASS — the objective explicitly includes resource leaks that lead to partial exports or silent data loss under sustained operation.
6. **Test correctness**: PASS — the PoC does not rely on mocks or internal-only APIs. I also reran it with aggressive GC (`GOGC=1`), and it still leaked 88 descriptors, so the result is not explained away by routine garbage collection during the short run.
7. **Alternative explanations**: PASS — Go's `os.File` finalizer is a partial mitigation, not an exoneration. The runtime source confirms a finalizer may eventually close descriptors, but the observed accumulation under default and aggressive GC still demonstrates unbounded growth over normal repeated batch execution.
8. **Novelty**: FAIL — this is an exact duplicate of already confirmed findings covering the same root cause and impact: `ai-summary/success/cli-commands/004-ledger-entry-changes-leaks-file-descriptors.md.gh-published`, `ai-summary/success/external-io/004-ledger-entry-changes-leaks-json-file-descriptors.md.gh-published`, and `ai-summary/success/data-integrity/014-ledger-entry-changes-leaks-file-descriptors.md.gh-published`.

### Rejection Reason

The filing describes a real bug, but it is not novel. The same `exportTransformedData()` / `MustOutFile()` descriptor leak has already been independently confirmed and published multiple times under other subsystem labels.

### Failed Checks

- Check 8 — novelty / duplicate detection
