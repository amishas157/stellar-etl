# 014: `export_ledger_entry_changes` leaves JSON writers open until GC

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes` opens one JSON writer per exported resource and never explicitly closes it inside `exportTransformedData()`. Each batch therefore leaks an extra writer file descriptor per resource until Go's garbage collector runs, and any close-time writeback error is discarded by runtime finalization instead of being surfaced to the exporter.

## Root Cause

`exportTransformedData()` calls `MustOutFile()` and `ExportEntry()` for every resource, but it never calls `outFile.Close()` before moving on. A separate helper bug in `createOutputFile()` already leaks one descriptor per new file for all callers; the command-specific defect here is the additional unclosed writer handle retained by `export_ledger_entry_changes`.

## Reproduction

Run the production `exportTransformedData()` path with several resource types and GC disabled. Compared with a control that follows the sibling-command pattern (`MustOutFile` -> `ExportEntry` -> explicit `Close`), the ledger-entry-changes path leaks one additional descriptor per resource. After forcing GC, those extra descriptors disappear, showing cleanup depends on finalizers rather than explicit `Close()`.

## Affected Code

- `cmd/export_ledger_entry_changes.go:304-373` — opens each resource file, writes rows, and never closes the JSON writer before returning or uploading.
- `cmd/command_utils.go:31-52` — `MustOutFile()` returns an open `*os.File` writer.
- `cmd/command_utils.go:55-86` — `ExportEntry()` writes JSON bytes directly to that writer.
- `cmd/command_utils.go:19-25` — separate shared baseline leak: `createOutputFile()` calls `os.Create()` without closing the returned file.

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportTransformedDataMissingCloseIsolated` and `TestExportTransformedDataLeakedFDsRecoveredOnlyByGC`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go test ./cmd/... -run 'TestExportTransformedData(MissingCloseIsolated|LeakedFDsRecoveredOnlyByGC)$' -v`.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"runtime"
	"runtime/debug"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// TestExportTransformedDataMissingCloseIsolated demonstrates that
// exportTransformedData leaks one EXTRA file descriptor per resource type
// because it never calls outFile.Close(), beyond the baseline FD leak in
// createOutputFile/MustOutFile that affects all callers.
//
// The test uses a control that mimics the sibling-command pattern
// (MustOutFile -> ExportEntry -> Close) to establish the baseline leak,
// then compares against exportTransformedData (MustOutFile -> ExportEntry -> no Close)
// to prove the incremental FD cost of the missing Close.
func TestExportTransformedDataMissingCloseIsolated(t *testing.T) {
	// Disable GC so os.File finalizers don't reclaim leaked FDs during measurement.
	oldGCPercent := debug.SetGCPercent(-1)
	defer debug.SetGCPercent(oldGCPercent)
	runtime.GC()
	runtime.GC()

	numFiles := 3

	// --- Control: sibling-command pattern (MustOutFile + ExportEntry + explicit Close) ---
	controlDir := t.TempDir()
	fdsBefore := countOpenFDs(t)

	for i := 0; i < numFiles; i++ {
		path := filepath.Join(controlDir, exportFilename(1, 11, []string{"accounts", "offers", "trustlines"}[i]))
		outFile := MustOutFile(path)
		_, err := ExportEntry(transform.AccountOutput{AccountID: "GABC", Balance: 100.0}, outFile, nil)
		if err != nil {
			t.Fatalf("ExportEntry failed: %v", err)
		}
		outFile.Close() // sibling commands do this
	}

	fdsAfterControl := countOpenFDs(t)
	controlLeak := fdsAfterControl - fdsBefore

	t.Logf("Control (with Close): %d FDs leaked for %d files (baseline from createOutputFile; before=%d, after=%d)",
		controlLeak, numFiles, fdsBefore, fdsAfterControl)

	// --- Bug path: exportTransformedData (MustOutFile + ExportEntry + NO Close) ---
	runtime.GC()
	runtime.GC()
	bugDir := t.TempDir()
	fdsBeforeBug := countOpenFDs(t)

	transformedOutput := map[string][]interface{}{
		"accounts":   {transform.AccountOutput{AccountID: "GABC", Balance: 100.0}},
		"offers":     {transform.OfferOutput{SellerID: "GABC", OfferID: 1}},
		"trustlines": {transform.TrustlineOutput{AccountID: "GABC"}},
	}

	err := exportTransformedData(1, 10, bugDir, bugDir, transformedOutput, "", "", "", nil, false)
	if err != nil {
		t.Fatalf("exportTransformedData returned error: %v", err)
	}

	fdsAfterBug := countOpenFDs(t)
	bugLeak := fdsAfterBug - fdsBeforeBug

	t.Logf("Bug path (no Close): %d FDs leaked for %d resources (before=%d, after=%d)",
		bugLeak, numFiles, fdsBeforeBug, fdsAfterBug)

	// The bug path should leak MORE FDs per resource than the control.
	// Control leaks N FDs from createOutputFile's os.Create.
	// Bug path leaks those same N FDs PLUS N more from the unclosed outFile.
	incrementalLeak := bugLeak - controlLeak
	if incrementalLeak < numFiles {
		t.Fatalf("Expected at least %d incremental FDs from missing outFile.Close() "+
			"(controlLeak=%d, bugLeak=%d, incremental=%d). "+
			"The bug may have been fixed.",
			numFiles, controlLeak, bugLeak, incrementalLeak)
	}

	t.Logf("BUG CONFIRMED: exportTransformedData leaks %d extra FDs beyond the createOutputFile baseline "+
		"(%d incremental for %d resources = ~%d extra per resource)",
		incrementalLeak, incrementalLeak, numFiles, incrementalLeak/numFiles)
	t.Logf("Sibling commands avoid this by calling outFile.Close() before MaybeUpload()")
}

// TestExportTransformedDataLeakedFDsRecoveredOnlyByGC shows that the extra
// FDs from the missing Close in exportTransformedData are only recovered
// when Go's GC runs finalizers, proving the lifecycle is accidentally
// delegated to the GC rather than handled by explicit cleanup.
// This means close-time I/O errors are silently discarded by the runtime.
func TestExportTransformedDataLeakedFDsRecoveredOnlyByGC(t *testing.T) {
	// Phase 1: Leak FDs (with GC disabled)
	oldGCPercent := debug.SetGCPercent(-1)
	defer debug.SetGCPercent(oldGCPercent)
	runtime.GC()
	runtime.GC()

	tmpDir := t.TempDir()
	fdsBefore := countOpenFDs(t)

	transformedOutput := map[string][]interface{}{
		"accounts":   {transform.AccountOutput{AccountID: "GABC", Balance: 100.0}},
		"offers":     {transform.OfferOutput{SellerID: "GABC", OfferID: 1}},
		"trustlines": {transform.TrustlineOutput{AccountID: "GABC"}},
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

	// Phase 2: Re-enable GC and force collection + finalizers.
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
	t.Logf("BUG CONFIRMED: exportTransformedData relies on GC finalizers for file cleanup - close-time I/O errors are silently discarded")
}

// countOpenFDs returns the number of open file descriptors for the current process.
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

## Expected vs Actual Behavior

- **Expected**: The exporter should explicitly close each JSON output file after writing, so descriptor usage stays bounded and close-time errors are reported to the caller.
- **Actual**: `export_ledger_entry_changes` leaves one extra writer descriptor open per resource until GC finalizers run, so descriptor usage grows across batches and close-time errors are never surfaced by the command.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the real `exportTransformedData()` implementation and measures the incremental FD leak against an explicit-close control.
2. Realistic preconditions: YES — the command processes multiple resource outputs per batch and can run continuously, so repeated descriptor growth is a normal production scenario.
3. Bug vs by-design: BUG — sibling exporters explicitly close their JSON writers, and no comment or contract indicates `export_ledger_entry_changes` should delegate cleanup to GC.
4. Final severity: Medium — this is operational correctness: later batches can fail at the OS file-descriptor limit, and close-time writeback failures are silently lost.
5. In scope: YES — it is a concrete production code path with measurable export reliability impact.
6. Test correctness: CORRECT — the PoC avoids the shared `createOutputFile()` leak by using a control path, then isolates the extra leak attributable to the missing `outFile.Close()`.
7. Alternative explanations: NONE — the baseline helper leak explains one FD per file, but the control-vs-bug comparison isolates the additional command-specific leak.
8. Novelty: NOVEL

## Suggested Fix

Call `outFile.Close()` in `exportTransformedData()` after the JSON write loop and before any upload, and return any close error. Separately, close the temporary handle created by `createOutputFile()` so every caller stops leaking the helper-level descriptor as well.
