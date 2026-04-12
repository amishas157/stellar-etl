# 021: Ledger entry batches leak file descriptors

**Date**: 2026-04-12
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes` opens a fresh JSON output file for each `(batch, resource)` pair and never closes it. On new paths, `MustOutFile()` also leaks the extra descriptor created by `os.Create()`, so repeated batch exports accumulate two live descriptors per file and can terminate later batches at the process open-file limit.

## Root Cause

`cmd.exportTransformedData()` calls `MustOutFile(path)` for every resource in every batch, writes rows, then returns to the next iteration without any `outFile.Close()`. `cmd.MustOutFile()` compounds that by calling `createOutputFile()`, whose `os.Create()` result is discarded without being closed when the path does not already exist.

## Reproduction

During normal operation, this manifests on long `export_ledger_entry_changes` runs with small batches or multiple enabled resource types. As the command creates more per-batch output files, descriptors accumulate until a later open hits `EMFILE`, at which point the command fatals and the remaining batch artifacts are never produced.

## Affected Code

- `cmd/command_utils.go:createOutputFile:19-25` — opens a new file with `os.Create()` and drops the returned handle
- `cmd/command_utils.go:MustOutFile:31-52` — opens the same path again and returns a second handle that callers must close
- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-376` — writes batch/resource JSON output without closing the returned file

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportTransformedDataLeaksFileDescriptors`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

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
	names, err := dir.Readdirnames(-1)
	dir.Close()
	if err != nil {
		t.Fatalf("cannot read %s: %v", fdDir, err)
	}
	return len(names)
}

func TestExportTransformedDataLeaksFileDescriptors(t *testing.T) {
	tmpDir := t.TempDir()
	folderPath := filepath.Join(tmpDir, "json")
	parquetFolderPath := filepath.Join(tmpDir, "parquet")

	dummyEntry := transform.TtlOutput{
		KeyHash:            "abc123",
		LiveUntilLedgerSeq: 100,
		LastModifiedLedger: 50,
	}
	transformedOutput := map[string][]interface{}{
		"ttl": {dummyEntry},
	}

	const iterations = 50

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
			"", "", "",
			nil,
			false,
		)
		if err != nil {
			t.Fatalf("exportTransformedData iteration %d failed: %v", i, err)
		}
	}

	fdAfter := countOpenFDs(t)
	t.Logf("FDs after %d iterations: %d", iterations, fdAfter)

	leaked := fdAfter - fdBefore
	t.Logf("Leaked FDs: %d (expected at least %d)", leaked, iterations)

	if leaked < iterations {
		t.Errorf("expected at least %d leaked FDs, but only found %d", iterations, leaked)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Each batch/resource output file should be closed promptly after writing so long-running exports can emit every requested artifact.
- **Actual**: JSON output descriptors accumulate across batches, and sufficiently long runs can stop early once file opens hit the OS limit.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC repeatedly calls the real `exportTransformedData()` path with distinct output filenames and measures process descriptor growth before and after.
2. Realistic preconditions: YES — `export_ledger_entry_changes` normally creates a fresh file per batch/resource combination, so repeated opens are part of ordinary bounded and continuous exports.
3. Bug vs by-design: BUG — sibling one-shot exporters explicitly close their JSON writers, and nothing in this command documents GC/finalizers as the resource-management strategy.
4. Final severity: Medium — the effect is partial export/data loss once later opens fail, not direct financial field corruption.
5. In scope: YES — the objective explicitly includes operational correctness bugs that silently truncate or abort exports under sustained operation.
6. Test correctness: CORRECT — it uses production code, no mocks, and a separate FD count instead of asserting on state the test itself fabricated.
7. Alternative explanations: NONE THAT EXONERATE THE BUG — an uncached rerun with `GOGC=1` still leaked 100 descriptors over 50 iterations, so routine garbage collection does not prevent the accumulation observed here.
8. Novelty: NOT ASSESSED HERE — duplicate handling is orchestrator-managed; this review confirms the bug exists and is in scope.

## Suggested Fix

Close `outFile` on every `exportTransformedData()` iteration before moving to the next resource, eliminate the redundant leaked `os.Create()` handle in `createOutputFile()`, and close the upload reader in `UploadTo()` so repeated batch exports do not rely on GC to release descriptors.
