# 004: `export_ledger_entry_changes` leaks JSON file descriptors

**Date**: 2026-04-10
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes` opens a fresh JSON output file for every `(batch, resource)` pair and never closes the returned `*os.File`. In long-running or continuous exports, those live descriptors accumulate until the process reaches the OS open-file limit and later batches stop exporting even though earlier files looked valid.

## Root Cause

`exportTransformedData()` calls `MustOutFile(path)` inside its per-resource loop and then writes entries and optionally uploads the path, but it never calls `outFile.Close()`. I also verified that `createOutputFile()` leaks an extra descriptor the first time it creates a new path, but the `exportTransformedData()` leak remains independently reproducible even when every target file is pre-created before the test runs.

## Reproduction

Any normal `export_ledger_entry_changes` run over many batches reaches this path because the command derives a distinct filename from each batch range and resource name. In continuous mode (`end-ledger=0`) or a large bounded export with several enabled resource types, the command keeps opening new JSON files and leaving them live until GC happens to reclaim them, so sustained runs can hit the process file-descriptor limit and stop producing later batch files.

## Affected Code

- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-376` — opens one JSON output file per resource and never closes it
- `cmd/command_utils.go:MustOutFile:31-52` — returns a live `*os.File` descriptor for each export path
- `cmd/command_utils.go:createOutputFile:19-28` — leaks an additional descriptor on first creation of a new path, but is not required to reproduce the `exportTransformedData()` leak

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportTransformedDataLeaksFileDescriptors`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `cmd/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./cmd/... -run TestExportTransformedDataLeaksFileDescriptors -v`
  4. Observe that 20 batches x 5 resource types leave 100 extra live descriptors even though the output files were pre-created.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"runtime"
	"runtime/debug"
	"testing"
)

func nextAvailableFD(t *testing.T) int {
	t.Helper()

	f, err := os.Open(os.DevNull)
	if err != nil {
		t.Fatalf("open %s: %v", os.DevNull, err)
	}
	fd := int(f.Fd())
	if err := f.Close(); err != nil {
		t.Fatalf("close %s: %v", os.DevNull, err)
	}

	return fd
}

func precreateExportFiles(t *testing.T, folderPath string, iterations int, resourceTypes []string) {
	t.Helper()

	for i := 0; i < iterations; i++ {
		start := uint32(i * 100)
		end := uint32((i+1)*100 - 1)
		for _, resource := range resourceTypes {
			path := filepath.Join(folderPath, exportFilename(start, end+1, resource))
			if err := os.MkdirAll(filepath.Dir(path), os.ModePerm); err != nil {
				t.Fatalf("mkdir %s: %v", filepath.Dir(path), err)
			}
			f, err := os.Create(path)
			if err != nil {
				t.Fatalf("precreate %s: %v", path, err)
			}
			if err := f.Close(); err != nil {
				t.Fatalf("close precreated file %s: %v", path, err)
			}
		}
	}
}

func makeExportOutput(resourceTypes []string) map[string][]interface{} {
	out := make(map[string][]interface{}, len(resourceTypes))
	for _, resource := range resourceTypes {
		out[resource] = []interface{}{
			map[string]string{
				"id":   "test",
				"type": resource,
			},
		}
	}

	return out
}

func TestExportTransformedDataLeaksFileDescriptors(t *testing.T) {
	tmpDir := t.TempDir()
	parquetDir := t.TempDir()
	resourceTypes := []string{"accounts", "offers", "trustlines", "ttl", "contract_data"}

	const iterations = 20
	precreateExportFiles(t, tmpDir, iterations, resourceTypes)

	runtime.GC()
	oldGCPercent := debug.SetGCPercent(-1)
	defer debug.SetGCPercent(oldGCPercent)

	fdBefore := nextAvailableFD(t)

	for i := 0; i < iterations; i++ {
		start := uint32(i * 100)
		end := uint32((i+1)*100 - 1)

		if err := exportTransformedData(
			start,
			end,
			tmpDir,
			parquetDir,
			makeExportOutput(resourceTypes),
			"", "", "",
			nil,
			false,
		); err != nil {
			t.Fatalf("exportTransformedData iteration %d: %v", i, err)
		}
	}

	fdAfter := nextAvailableFD(t)
	leaked := fdAfter - fdBefore
	expected := iterations * len(resourceTypes)

	t.Logf("FD before=%d after=%d leaked=%d expected=%d", fdBefore, fdAfter, leaked, expected)

	if leaked < expected {
		t.Fatalf("expected at least %d leaked descriptors from unclosed MustOutFile handles, got %d", expected, leaked)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Each JSON output file should be closed after its batch has been written (and uploaded, if enabled), so the number of live descriptors stays bounded over time.
- **Actual**: Each batch/resource export leaves another live descriptor behind; sustained runs eventually stop exporting later batches when the process hits the OS file-descriptor limit.

## Adversarial Review

1. Exercises claimed bug: YES — the corrected PoC calls the real `exportTransformedData()` path and pre-creates every output file so the observed 100 leaked descriptors come from the missing `Close()` on the returned `*os.File`, not from first-time file creation.
2. Realistic preconditions: YES — the command is explicitly designed for batched and continuous (`end-ledger=0`) exports, and each batch naturally produces fresh output filenames across multiple resource types.
3. Bug vs by-design: BUG — every sibling export command closes its JSON output file, while `export_ledger_entry_changes` is the lone outlier.
4. Final severity: Medium — the issue does not corrupt row contents, but it can silently halt future exports and lose later ledger-entry-change batches under sustained normal operation.
5. In scope: YES — this is a concrete production export path causing data loss from normal operation, not an attacker-driven resource exhaustion report.
6. Test correctness: CORRECT — I rewrote the original PoC to remove the `createOutputFile()` confounder; the final test isolates the specific missing-`Close()` claim and still reproduces the leak.
7. Alternative explanations: NONE — `createOutputFile()` has its own leak, but pre-creating every target file shows that `exportTransformedData()` independently leaks one descriptor per output file.
8. Novelty: NOVEL

## Suggested Fix

Call `outFile.Close()` on every `exportTransformedData()` path, including error paths, and remove the extra `os.Create()` leak in `createOutputFile()` so output file descriptors are always closed deterministically.
