# H005: `WriteParquet` drops footer and flush failures from `WriteStop`

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If parquet finalization fails while flushing row groups, writing column indexes, writing the footer, or closing the destination file, the export command should surface that failure and stop before claiming success or uploading the file. A truncated parquet file should never be treated as a completed export.

## Mechanism

`WriteParquet` defers both `writer.WriteStop()` and `parquetFile.Close()` but ignores their returned errors. In parquet-go, `WriteStop()` performs the final `Flush(true)` plus footer and index writes, so late `ENOSPC`, quota, or filesystem errors can happen after all per-row `writer.Write(...)` calls already succeeded. Those failures are silently discarded, and the caller continues as though the parquet file were valid.

## Trigger

Run any parquet-writing export on a filesystem that allows row writes but fails during footer finalization, such as a nearly full disk or quota-limited mount. The command will return from `WriteParquet` without error even though the parquet footer write failed and the output file is incomplete.

## Target Code

- `cmd/command_utils.go:WriteParquet:162-180` — deferred `WriteStop()` and `Close()` return values are ignored
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/writer/writer.go:129-202` — `WriteStop()` performs final flush and footer/index writes and returns I/O errors
- `cmd/export_assets.go:79-81` — representative caller assumes `WriteParquet` succeeded and proceeds to upload
- `cmd/export_contract_events.go:63-65` — another caller treats normal return as a completed parquet export

## Evidence

The only checked write path inside `WriteParquet` is the per-record `writer.Write(...)` loop. The deferred finalization steps are the ones that actually emit the parquet footer and trailer magic, and parquet-go explicitly returns errors from those writes, but the helper never captures or propagates them.

## Anti-Evidence

Mid-stream row-write failures are still handled correctly because the loop calls `cmdLogger.Fatal` immediately on `writer.Write(...)` errors. The missed corruption window is specifically the late finalization phase after all rows have already been accepted.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`WriteParquet` (cmd/command_utils.go:162-180) uses `defer writer.WriteStop()` and `defer parquetFile.Close()`, discarding both error returns. In Go, `defer f()` silently drops return values. The parquet-go `WriteStop()` method (writer.go:129-202) performs six distinct I/O operations—Flush, ColumnIndex writes, OffsetIndex writes, footer serialization, footer size, and trailing "PAR1" magic—each returning errors that propagate to the caller. Since `WriteParquet` has no return value and defers finalization without error capture, any I/O failure during these critical steps is invisible to the five callers that proceed to `MaybeUpload` immediately after.

### Code Paths Examined

- `cmd/command_utils.go:WriteParquet:162-180` — confirmed: `defer writer.WriteStop()` on line 173 discards the error; `defer parquetFile.Close()` on line 167 discards the error; function signature returns nothing
- `writer/writer.go:WriteStop:129-202` (parquet-go v1.6.2) — confirmed: performs `Flush(true)`, writes ColumnIndex buffers, writes OffsetIndex buffers, writes footer, writes footer size, writes "PAR1" trailer; each step returns error on I/O failure
- `local/local.go:Write` (parquet-go-source) — confirmed: delegates to `os.File.Write`, propagates errors
- `cmd/export_assets.go:80-81` — confirmed: calls `WriteParquet` then `MaybeUpload` with no error check between them
- `cmd/export_effects.go:67-68` — confirmed: same pattern
- `cmd/export_contract_events.go:64-65` — confirmed: same pattern
- `cmd/export_ledger_entry_changes.go:371-372` — confirmed: same pattern

### Findings

The bug is confirmed. There is a clear inconsistency in error handling within `WriteParquet`:
1. **Row writes**: `writer.Write()` errors trigger `cmdLogger.Fatal` — process terminates immediately.
2. **Finalization**: `writer.WriteStop()` errors are silently discarded via `defer`.
3. **File close**: `parquetFile.Close()` errors are silently discarded via `defer`.

Without a valid footer (written by `WriteStop`), a parquet file is structurally invalid and unreadable by any standard parquet reader (Spark, BigQuery, pyarrow). Five callers proceed to `MaybeUpload` after `WriteParquet` returns, potentially uploading a corrupt file to GCS.

This is distinct from existing findings:
- Success 001 covers upload-before-write ordering (different bug)
- Success 002 covers ExportEntry swallowing JSON write errors (different function, different I/O path)

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` or a new `cmd/write_parquet_poc_test.go`
- **Setup**: Create a mock `source.ParquetFile` implementation that accepts all `Write` calls during row writing but returns an error during the footer-phase writes (i.e., after N successful Write calls, start returning `os.ErrClosed` or a custom error). Alternatively, use a real file on a tmpfs with a small size limit.
- **Steps**: Call `WriteParquet` with valid data and the error-injecting file. Observe that the function returns without panic/fatal.
- **Assertion**: After `WriteParquet` returns, attempt to read the output file with a parquet reader (`parquet-go/reader`). The read should fail because the footer is missing or truncated. This proves the function silently produced an invalid file. Alternatively, a static analysis test can verify that `defer writer.WriteStop()` is used without error capture (similar to the source-scanning approach in the existing PoC for success/001).

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestWriteParquetIgnoresFinalizationErrors"
**Test Language**: Go

### Demonstration

The test proves the bug through three complementary demonstrations: (1) parquet-go's `WriteStop()` returns real I/O errors when the underlying file is closed before finalization, confirming that footer-phase failures are a genuine possibility; (2) Go AST analysis of `command_utils.go` confirms that `WriteParquet` has no error return value and uses bare `defer writer.WriteStop()` / `defer parquetFile.Close()` which silently discard errors; (3) a failed `WriteStop` produces a 4-byte file containing only the PAR1 header magic — structurally invalid and unreadable — while a valid file for identical data is 1466 bytes with a proper footer. Since `WriteParquet` cannot report this failure, all five export callers proceed to `MaybeUpload` with a corrupt artifact.

### Test Body

```go
package cmd

import (
	"go/ast"
	"go/parser"
	"go/token"
	"os"
	"path/filepath"
	"runtime"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/writer"
)

// TestWriteParquetIgnoresFinalizationErrors demonstrates that WriteParquet
// silently drops errors from the parquet footer/finalization phase.
//
// WriteParquet uses `defer writer.WriteStop()` which discards the returned
// error, and the function itself has no error return. This means callers
// (all 5 export commands) proceed to MaybeUpload even if the parquet file
// is structurally invalid (missing footer).
func TestWriteParquetIgnoresFinalizationErrors(t *testing.T) {

	// Part 1: Prove that parquet-go WriteStop() DOES return errors when the
	// underlying file is closed before finalization. This uses the exact same
	// library calls as WriteParquet.
	t.Run("WriteStop returns error on closed file", func(t *testing.T) {
		tmpDir := t.TempDir()
		path := filepath.Join(tmpDir, "test.parquet")

		pf, err := local.NewLocalFileWriter(path)
		if err != nil {
			t.Fatalf("NewLocalFileWriter failed: %v", err)
		}

		pw, err := writer.NewParquetWriter(pf, new(transform.AssetOutputParquet), 1)
		if err != nil {
			t.Fatalf("NewParquetWriter failed: %v", err)
		}

		record := transform.AssetOutputParquet{
			AssetCode:   "XLM",
			AssetIssuer: "GABC",
			AssetType:   "native",
			AssetID:     123,
		}
		if err := pw.Write(record); err != nil {
			t.Fatalf("Write failed: %v", err)
		}

		// Close the underlying file BEFORE finalization — simulates the
		// class of I/O failures (ENOSPC, EIO, quota) that can occur after
		// all row writes succeed but before the footer is flushed.
		pf.Close()

		// WriteStop now attempts to flush, write column indexes, offset
		// indexes, footer, footer size, and PAR1 trailer to a closed file.
		err = pw.WriteStop()
		if err == nil {
			t.Fatal("Expected WriteStop to return an error on a closed file, but it succeeded")
		}
		t.Logf("CONFIRMED: WriteStop returned error on closed file: %v", err)
		t.Log("In WriteParquet, this error is silently discarded by: defer writer.WriteStop()")
	})

	// Part 2: Structural proof — parse the source code and verify that:
	// (a) WriteParquet has NO error return value
	// (b) WriteStop and Close are called via defer without error capture
	t.Run("WriteParquet structurally discards finalization errors", func(t *testing.T) {
		// Locate command_utils.go relative to this test file
		_, thisFile, _, ok := runtime.Caller(0)
		if !ok {
			t.Fatal("runtime.Caller failed")
		}
		srcPath := filepath.Join(filepath.Dir(thisFile), "command_utils.go")

		fset := token.NewFileSet()
		f, err := parser.ParseFile(fset, srcPath, nil, 0)
		if err != nil {
			t.Fatalf("Failed to parse command_utils.go: %v", err)
		}

		found := false
		for _, decl := range f.Decls {
			fn, ok := decl.(*ast.FuncDecl)
			if !ok || fn.Name.Name != "WriteParquet" {
				continue
			}
			found = true

			// (a) Verify no return type — callers can never learn about errors
			if fn.Type.Results != nil && len(fn.Type.Results.List) > 0 {
				t.Fatal("WriteParquet now has a return value — hypothesis outdated")
			}
			t.Log("CONFIRMED: WriteParquet has no return value — errors cannot propagate")

			// (b) Find defer statements that discard error-returning calls
			deferredCalls := []string{}
			ast.Inspect(fn.Body, func(n ast.Node) bool {
				ds, ok := n.(*ast.DeferStmt)
				if !ok {
					return true
				}
				if sel, ok := ds.Call.Fun.(*ast.SelectorExpr); ok {
					name := sel.Sel.Name
					if name == "WriteStop" || name == "Close" {
						deferredCalls = append(deferredCalls, name)
						t.Logf("CONFIRMED: defer %s() discards its error return", name)
					}
				}
				return true
			})

			if len(deferredCalls) == 0 {
				t.Fatal("No deferred WriteStop/Close calls found — source may have changed")
			}
		}
		if !found {
			t.Fatal("WriteParquet function not found in command_utils.go")
		}
	})

	// Part 3: Establish that a successful WriteParquet produces a valid file
	// with proper size and PAR1 footer, while a failed WriteStop produces a
	// truncated file (only 4-byte header). Combined with Part 1, this proves
	// that when WriteStop fails the footer is missing and the file is corrupt
	// — yet WriteParquet returns as if everything worked.
	t.Run("failed WriteStop produces truncated file missing footer", func(t *testing.T) {
		tmpDir := t.TempDir()

		// Write a valid parquet file via the production WriteParquet function
		goodPath := filepath.Join(tmpDir, "good.parquet")
		data := []transform.SchemaParquet{
			transform.AssetOutput{
				AssetCode:   "XLM",
				AssetIssuer: "GABC",
				AssetType:   "native",
				AssetID:     123,
			},
		}
		WriteParquet(data, goodPath, new(transform.AssetOutputParquet))

		goodStat, err := os.Stat(goodPath)
		if err != nil {
			t.Fatalf("Failed to stat good file: %v", err)
		}
		goodSize := goodStat.Size()
		t.Logf("Valid parquet file: %d bytes", goodSize)

		// Verify the valid file ends with PAR1 magic
		goodData, err := os.ReadFile(goodPath)
		if err != nil {
			t.Fatalf("Failed to read good file: %v", err)
		}
		if string(goodData[len(goodData)-4:]) != "PAR1" {
			t.Fatal("Valid parquet file missing PAR1 footer")
		}
		t.Log("CONFIRMED: Valid parquet file has PAR1 footer")

		// Now reproduce the failure scenario: close file before WriteStop
		badPath := filepath.Join(tmpDir, "bad.parquet")
		pf, err := local.NewLocalFileWriter(badPath)
		if err != nil {
			t.Fatalf("NewLocalFileWriter failed: %v", err)
		}
		pw, err := writer.NewParquetWriter(pf, new(transform.AssetOutputParquet), 1)
		if err != nil {
			t.Fatalf("NewParquetWriter failed: %v", err)
		}
		record := transform.AssetOutputParquet{
			AssetCode:   "XLM",
			AssetIssuer: "GABC",
			AssetType:   "native",
			AssetID:     123,
		}
		if err := pw.Write(record); err != nil {
			t.Fatalf("Write failed: %v", err)
		}
		pf.Close()
		_ = pw.WriteStop() // error discarded — same as WriteParquet's defer

		badStat, err := os.Stat(badPath)
		if err != nil {
			t.Fatalf("Failed to stat bad file: %v", err)
		}
		badSize := badStat.Size()

		// The bad file should be just the 4-byte PAR1 header (no row data,
		// no column indexes, no footer). A valid file for the same data is
		// ~1400+ bytes.
		if badSize >= goodSize {
			t.Fatalf("Expected truncated file, got %d bytes (good=%d)", badSize, goodSize)
		}

		// The minimum valid parquet file needs header(4) + footer_meta +
		// footer_size(4) + trailer_magic(4) = well over 12 bytes.
		// A 4-byte file is just the header magic — structurally invalid.
		const minValidParquetSize = 12
		if badSize >= minValidParquetSize {
			t.Fatalf("Bad file is %d bytes — expected < %d for a truncated file", badSize, minValidParquetSize)
		}

		t.Logf("CONFIRMED: Failed WriteStop produced %d-byte file (vs %d-byte valid file)", badSize, goodSize)
		t.Log("The corrupt file has only the PAR1 header — no row data, no footer")
		t.Log("BUG: WriteParquet's `defer writer.WriteStop()` silently discards this error")
		t.Log("     All 5 export callers proceed to MaybeUpload with a corrupt parquet artifact")
	})
}
```

### Test Output

```
=== RUN   TestWriteParquetIgnoresFinalizationErrors
=== RUN   TestWriteParquetIgnoresFinalizationErrors/WriteStop_returns_error_on_closed_file
    data_integrity_poc_test.go:64: CONFIRMED: WriteStop returned error on closed file: write .../test.parquet: file already closed
    data_integrity_poc_test.go:65: In WriteParquet, this error is silently discarded by: defer writer.WriteStop()
=== RUN   TestWriteParquetIgnoresFinalizationErrors/WriteParquet_structurally_discards_finalization_errors
    data_integrity_poc_test.go:97: CONFIRMED: WriteParquet has no return value — errors cannot propagate
    data_integrity_poc_test.go:110: CONFIRMED: defer Close() discards its error return
    data_integrity_poc_test.go:110: CONFIRMED: defer WriteStop() discards its error return
=== RUN   TestWriteParquetIgnoresFinalizationErrors/failed_WriteStop_produces_truncated_file_missing_footer
    data_integrity_poc_test.go:150: Valid parquet file: 1466 bytes
    data_integrity_poc_test.go:160: CONFIRMED: Valid parquet file has PAR1 footer
    data_integrity_poc_test.go:205: CONFIRMED: Failed WriteStop produced 4-byte file (vs 1466-byte valid file)
    data_integrity_poc_test.go:206: The corrupt file has only the PAR1 header — no row data, no footer
    data_integrity_poc_test.go:207: BUG: WriteParquet's `defer writer.WriteStop()` silently discards this error
    data_integrity_poc_test.go:208:      All 5 export callers proceed to MaybeUpload with a corrupt parquet artifact
--- PASS: TestWriteParquetIgnoresFinalizationErrors (0.01s)
    --- PASS: TestWriteParquetIgnoresFinalizationErrors/WriteStop_returns_error_on_closed_file (0.00s)
    --- PASS: TestWriteParquetIgnoresFinalizationErrors/WriteParquet_structurally_discards_finalization_errors (0.00s)
    --- PASS: TestWriteParquetIgnoresFinalizationErrors/failed_WriteStop_produces_truncated_file_missing_footer (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	1.873s
```
