# 004: `export_ledger_entry_changes` leaks file descriptors across batches

**Date**: 2026-04-10
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes` opens a fresh JSON output file for every `(batch, resource)` pair and never closes it. It also leaks an extra descriptor when first creating each new output path, so long-running exports accumulate live file handles until later batches stop exporting once the process reaches the OS open-file limit.

## Root Cause

`exportTransformedData()` calls `MustOutFile(path)` inside its per-resource loop, writes JSON rows, and returns without ever calling `outFile.Close()`. Separately, `createOutputFile()` calls `os.Create(filepath)` when the path is absent and discards the returned `*os.File`, leaking another descriptor for each newly created batch file.

## Reproduction

This path is exercised during normal `export_ledger_entry_changes` operation because the command builds a distinct output filename from each batch range and resource name, then calls `exportTransformedData()` once per batch. Continuous mode is the easiest trigger, but any sufficiently large multi-batch export with several enabled resource types will leak descriptors even without cloud upload, because every batch writes new filenames.

## Affected Code

- `cmd/export_ledger_entry_changes.go:Run:274-285` — invokes `exportTransformedData()` once per processed batch
- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-376` — opens one JSON output file per resource and never closes it
- `cmd/command_utils.go:MustOutFile:31-52` — returns a live `*os.File` for each export path
- `cmd/command_utils.go:createOutputFile:19-28` — leaks a descriptor when first creating a missing output file

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportTransformedDataLeaksFileDescriptors`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `cmd/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./cmd/... -run TestExportTransformedDataLeaksFileDescriptors -v`
  4. Observe descriptor growth (`initial_fds=5 final_fds=185 leaked=180` in this review) after 30 iterations over 3 resource types.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"runtime"
	"runtime/debug"
	"syscall"
	"testing"
)

func countOpenFDs(t *testing.T) int {
	t.Helper()

	count := 0
	for fd := 0; fd < 4096; fd++ {
		_, _, errno := syscall.Syscall(syscall.SYS_FCNTL, uintptr(fd), uintptr(syscall.F_GETFD), 0)
		if errno == 0 {
			count++
		}
	}

	return count
}

func TestExportTransformedDataLeaksFileDescriptors(t *testing.T) {
	originalGCPercent := debug.SetGCPercent(-1)
	defer debug.SetGCPercent(originalGCPercent)

	tmpDir := t.TempDir()
	outputFolder := filepath.Join(tmpDir, "output")
	parquetFolder := filepath.Join(tmpDir, "parquet")
	if err := os.MkdirAll(outputFolder, 0o755); err != nil {
		t.Fatal(err)
	}
	if err := os.MkdirAll(parquetFolder, 0o755); err != nil {
		t.Fatal(err)
	}

	runtime.GC()
	initialFDs := countOpenFDs(t)

	const iterations = 30
	resourceTypes := []string{"accounts", "offers", "trustlines"}

	for i := 0; i < iterations; i++ {
		transformedOutput := map[string][]interface{}{}
		for _, resource := range resourceTypes {
			transformedOutput[resource] = []interface{}{}
		}

		start := uint32(i * 10)
		end := start + 9

		if err := exportTransformedData(
			start,
			end,
			outputFolder,
			parquetFolder,
			transformedOutput,
			"",
			"",
			"",
			nil,
			false,
		); err != nil {
			t.Fatalf("iteration %d failed: %v", i, err)
		}
	}

	finalFDs := countOpenFDs(t)
	leaked := finalFDs - initialFDs
	minExpectedLeak := iterations * len(resourceTypes)

	t.Logf("initial_fds=%d final_fds=%d leaked=%d", initialFDs, finalFDs, leaked)

	if leaked < minExpectedLeak {
		t.Fatalf("expected at least %d leaked file descriptors, got %d (initial=%d final=%d)", minExpectedLeak, leaked, initialFDs, finalFDs)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Each batch/resource output file should be closed after export so the number of live descriptors remains bounded over time.
- **Actual**: The export path leaks file descriptors on every batch/resource pair; in the reproduced run, 30 iterations over 3 resources leaked 180 descriptors and the count grew exactly with repeated batch execution.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the real `exportTransformedData()` helper used by the CLI and reproduces descriptor growth on repeated batch exports.
2. Realistic preconditions: YES — this is the normal batched export path, and cloud upload is not required because batch-specific filenames are unique even in local-only runs.
3. Bug vs by-design: BUG — the sibling exporters close their JSON files, while `export_ledger_entry_changes` is the lone outlier.
4. Final severity: Medium — this is operational correctness breakage that can halt later exports and lose subsequent ledger-entry-change batches under sustained normal operation.
5. In scope: YES — the objective explicitly includes resource leaks that cause data loss under sustained operation.
6. Test correctness: CORRECT — the PoC uses production code, disables GC to prevent finalizer cleanup, and counts open descriptors via `fcntl` rather than by asserting a self-fulfilling condition.
7. Alternative explanations: NONE — the observed 180 leaked descriptors match the traced `30 iterations x 3 resources x 2 leaked descriptors` mechanism exactly.
8. Novelty: NOT ASSESSED — duplicate handling is external to this final review step.

## Suggested Fix

Close `outFile` deterministically on every `exportTransformedData()` path, including error paths, and close the file returned by `os.Create()` inside `createOutputFile()` instead of discarding it.
