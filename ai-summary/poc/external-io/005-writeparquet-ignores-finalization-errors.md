# H005: `WriteParquet()` treats footer/close failures as success and can upload truncated parquet

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If Parquet finalization fails while flushing row groups, writing indexes/footer, or closing the file handle, the export command should surface that error and avoid reporting success or uploading the malformed artifact.

## Mechanism

`WriteParquet()` defers both `parquetFile.Close()` and `writer.WriteStop()` without inspecting their return values. In `parquet-go`, `WriteStop()` is the step that flushes pending pages and writes the footer/index blocks; a late filesystem error at that stage is silently discarded, after which callers continue as though the Parquet file is complete and may upload a truncated or unreadable file.

## Trigger

Run any `--write-parquet` export on storage that starts failing only during final flush, footer, or close — for example, a nearly full disk that accepts row writes but errors once `WriteStop()` serializes the footer and trailing metadata.

## Target Code

- `cmd/command_utils.go:162-180` — defers `parquetFile.Close()` and `writer.WriteStop()` without checking either error
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/writer/writer.go:128-202` — `WriteStop()` returns errors from `Flush()` and footer/index `PFile.Write()` calls
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/example/local_flat.go:51-60` — upstream examples explicitly check the `WriteStop()` error before declaring success

## Evidence

The production helper only handles errors from `writer.NewParquetWriter()` and per-record `writer.Write()`. The library itself documents the real finalization contract in code: `WriteStop()` returns `error`, and upstream examples treat that return as mandatory to check.

## Anti-Evidence

Immediate write-path failures are still surfaced because `writer.Write()` is checked inside the loop. This hypothesis only concerns late failures that happen after the last row was accepted but before the file becomes a valid finished Parquet artifact.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`WriteParquet()` in `cmd/command_utils.go:162-180` uses `defer writer.WriteStop()` and `defer parquetFile.Close()`, discarding both error returns. The parquet-go library's `WriteStop()` method (`writer/writer.go:128-202`) performs substantial finalization work — flushing remaining pages via `Flush(true)`, serializing column indexes, offset indexes, the Thrift-encoded footer, footer size, and the trailing `PAR1` magic bytes — returning errors from every `PFile.Write()` call. Because the defer discards these errors, `WriteParquet()` returns as though the file is complete. At least four export commands (`export_assets`, `export_effects`, `export_contract_events`, `export_ledger_entry_changes`) then call `MaybeUpload()` immediately after, uploading a potentially truncated parquet file to GCS.

### Code Paths Examined

- `cmd/command_utils.go:162-180` — `WriteParquet()` defers `WriteStop()` and `Close()` without capturing error returns; function signature is `void` so cannot propagate errors
- `parquet-go/writer/writer.go:128-202` — `WriteStop()` calls `Flush(true)` then writes column indexes, offset indexes, footer, footer size, and magic bytes, returning error at each stage
- `parquet-go/writer/writer.go:246-300` — `Flush()` calls `flushObjs()` then writes row group chunk data with error checking
- `cmd/export_effects.go:67-68` — calls `WriteParquet()` then `MaybeUpload()` in sequence
- `cmd/export_assets.go:80-81` — same Write-then-Upload pattern
- `cmd/export_contract_events.go:64-65` — same pattern
- `cmd/export_ledger_entry_changes.go:371-372` — same pattern

### Findings

The bug is real: Go's `defer` statement discards the return value of the deferred function call. Since `WriteParquet()` has no error return, there is no mechanism for callers to learn that finalization failed. The `WriteStop()` method in parquet-go performs the most critical part of parquet file construction — without the footer and trailing magic bytes, the file is unreadable by any parquet consumer. If the underlying filesystem returns an error during any of the ~6 write operations in `WriteStop()`, the error is silently dropped and the export continues to upload the corrupt artifact.

This is distinct from the previously confirmed finding `003-stale-parquet-upload-before-regeneration` (which concerns upload ordering) and `008-export-entry-swallows-write-errors` (which concerns JSON output error handling in `ExportEntry()`).

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (or create one if it doesn't exist)
- **Setup**: Create a mock `ParquetFile` implementation that returns `nil` from `Write()` during normal row writes but returns an error when `WriteStop()` calls its final `Write()` (e.g., track call count and fail on the Nth write). Alternatively, use a wrapper around `local.NewLocalFileWriter` that returns write errors after a byte threshold.
- **Steps**: Call `WriteParquet()` with a valid schema and a small dataset using the error-injecting file writer. Verify that `WriteParquet()` returns without panicking or fataling (confirming the error is swallowed).
- **Assertion**: After `WriteParquet()` returns, read the output file and verify it is NOT a valid parquet file (missing footer/magic bytes). This proves a caller would proceed to upload corrupt data.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestWriteParquetIgnoresFinalizationErrors"
**Test Language**: Go

### Demonstration

The test replicates the exact `defer writer.WriteStop()` pattern from `WriteParquet()` using a fault-injecting `ParquetFile` wrapper that fails on the 4th Write call (during finalization). The production pattern function returns nil (success) even though `WriteStop()` fails with "simulated disk full during finalization". The resulting file is 145 bytes and missing the required PAR1 footer, making it unreadable by any parquet consumer. A separate explicit call to `WriteStop()` with the same faulty file confirms it returns a non-nil error — proving the `defer` pattern silently discards it.

### Test Body

```go
package cmd

import (
	"errors"
	"os"
	"testing"

	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/source"
	"github.com/xitongsys/parquet-go/writer"
)

// faultInjectingFile wraps a real ParquetFile and injects write errors
// after a configurable number of Write calls, simulating disk-full during
// parquet finalization (WriteStop).
type faultInjectingFile struct {
	wrapped    source.ParquetFile
	writeCount int
	failAfterN int // return error starting at the Nth Write call
}

func (f *faultInjectingFile) Write(p []byte) (int, error) {
	f.writeCount++
	if f.writeCount >= f.failAfterN {
		return 0, errors.New("simulated disk full during finalization")
	}
	return f.wrapped.Write(p)
}

func (f *faultInjectingFile) Read(p []byte) (int, error)                   { return f.wrapped.Read(p) }
func (f *faultInjectingFile) Seek(offset int64, whence int) (int64, error) { return f.wrapped.Seek(offset, whence) }
func (f *faultInjectingFile) Close() error                                 { return f.wrapped.Close() }
func (f *faultInjectingFile) Open(name string) (source.ParquetFile, error) { return f.wrapped.Open(name) }
func (f *faultInjectingFile) Create(name string) (source.ParquetFile, error) {
	return f.wrapped.Create(name)
}

// simpleParquetRecord is a minimal parquet-tagged struct for testing.
type simpleParquetRecord struct {
	Name  string `parquet:"name=name, type=BYTE_ARRAY, convertedtype=UTF8"`
	Value int32  `parquet:"name=value, type=INT32"`
}

// writeParquetProductionPattern replicates the exact error-handling pattern
// from cmd/command_utils.go:162-180 (WriteParquet). It uses defer for both
// WriteStop and Close, discarding their error returns.
func writeParquetProductionPattern(pf source.ParquetFile, data []simpleParquetRecord) error {
	defer pf.Close()

	pw, err := writer.NewParquetWriter(pf, new(simpleParquetRecord), 1)
	if err != nil {
		return err
	}
	defer pw.WriteStop() // ERROR DISCARDED — this is the bug

	for _, record := range data {
		if err := pw.Write(record); err != nil {
			return err
		}
	}
	return nil // returns success even if WriteStop will fail
}

// TestWriteParquetIgnoresFinalizationErrors demonstrates that the production
// WriteParquet pattern (defer writer.WriteStop()) silently discards errors
// from parquet finalization, producing corrupt files without any error signal.
func TestWriteParquetIgnoresFinalizationErrors(t *testing.T) {
	// Create temp file for parquet output
	tmpFile, err := os.CreateTemp("", "poc-parquet-*.parquet")
	if err != nil {
		t.Fatal(err)
	}
	tmpPath := tmpFile.Name()
	tmpFile.Close()
	defer os.Remove(tmpPath)

	// Create a real local file writer
	realFile, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatal(err)
	}

	// Wrap with fault injection: fail writes starting at the 4th Write call.
	// The first few writes handle the parquet header and row data pages.
	// Writes during WriteStop (column indexes, offset indexes, footer, footer
	// size, PAR1 magic) will hit the injected error.
	faultyFile := &faultInjectingFile{
		wrapped:    realFile,
		failAfterN: 4,
	}

	records := []simpleParquetRecord{
		{Name: "test-record", Value: 42},
	}

	// Call the production pattern — it should return nil even when WriteStop fails
	err = writeParquetProductionPattern(faultyFile, records)
	if err != nil {
		t.Fatalf("Production pattern returned error during row writes: %v\n"+
			"Adjusting fault threshold — this means row writes consumed >= 4 Write calls.\n"+
			"Try increasing failAfterN.", err)
	}

	// Read the output file
	data, err := os.ReadFile(tmpPath)
	if err != nil {
		t.Fatal(err)
	}

	// Valid parquet files MUST end with the 4-byte magic "PAR1"
	hasValidFooter := len(data) >= 4 && string(data[len(data)-4:]) == "PAR1"

	if !hasValidFooter {
		// BUG CONFIRMED: WriteParquet returned success but the file is corrupt
		t.Logf("Bug confirmed: WriteParquet production pattern returned nil (success)")
		t.Logf("Output file is %d bytes and MISSING PAR1 footer — file is corrupt/unreadable", len(data))
		t.Logf("Any subsequent MaybeUpload() call would upload this corrupt file to GCS")

		// Also verify WriteStop would have returned an error if checked
		realFile2, err := local.NewLocalFileWriter(tmpPath)
		if err != nil {
			t.Fatal(err)
		}
		faultyFile2 := &faultInjectingFile{wrapped: realFile2, failAfterN: 4}
		pw2, err := writer.NewParquetWriter(faultyFile2, new(simpleParquetRecord), 1)
		if err != nil {
			t.Fatal(err)
		}
		_ = pw2.Write(simpleParquetRecord{Name: "test", Value: 1})
		writeStopErr := pw2.WriteStop()
		faultyFile2.Close()
		if writeStopErr != nil {
			t.Logf("WriteStop explicitly returns error: %v", writeStopErr)
			t.Logf("But 'defer writer.WriteStop()' discards this error — confirmed silent data corruption")
		}
	} else {
		t.Errorf("File has valid PAR1 footer (%d bytes) — fault injection threshold may be too high, "+
			"allowing WriteStop to complete. Increase failAfterN.", len(data))
	}
}
```

### Test Output

```
=== RUN   TestWriteParquetIgnoresFinalizationErrors
    data_integrity_poc_test.go:115: Bug confirmed: WriteParquet production pattern returned nil (success)
    data_integrity_poc_test.go:116: Output file is 145 bytes and MISSING PAR1 footer — file is corrupt/unreadable
    data_integrity_poc_test.go:117: Any subsequent MaybeUpload() call would upload this corrupt file to GCS
    data_integrity_poc_test.go:133: WriteStop explicitly returns error: simulated disk full during finalization
    data_integrity_poc_test.go:134: But 'defer writer.WriteStop()' discards this error — confirmed silent data corruption
--- PASS: TestWriteParquetIgnoresFinalizationErrors (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.734s
```
