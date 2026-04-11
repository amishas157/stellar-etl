# 012: `WriteParquet` ignores parquet finalization errors

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`WriteParquet` checks row-write failures but silently drops errors returned by the final `WriteStop()` and `Close()` calls. If the filesystem fails during parquet finalization, the helper returns normally and callers can keep or upload a truncated parquet artifact as if the export succeeded.

This is a real runtime failure mode, not just a style issue. `parquet-go` returns I/O errors from the footer/index/trailer writes performed in `WriteStop()`, and the helper has no return value or logging path for those late failures.

## Root Cause

`cmd.WriteParquet` defers both `parquetFile.Close()` and `writer.WriteStop()` without capturing their returned errors. Because deferred call results are discarded and the function itself returns nothing, late I/O failures during parquet finalization cannot reach the export commands that invoke it.

## Reproduction

Run any parquet-capable export on a filesystem that accepts initial output but fails during the footer/index/trailer writes at finalization time, such as a nearly full disk or a quota-limited mount. The row loop completes, `WriteParquet` returns normally, and the caller proceeds as though the parquet file were complete even though `WriteStop()` would have reported an I/O error and the artifact is truncated.

## Affected Code

- `cmd/command_utils.go:162-180` — `WriteParquet` defers `Close()` and `WriteStop()` and never checks their returned errors
- `cmd/export_assets.go:79-81` — treats normal `WriteParquet` return as a completed parquet export and immediately uploads
- `cmd/export_contract_events.go:63-65` — same post-`WriteParquet` success assumption on another command path

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestWriteParquetIgnoresFinalizationErrors`
- **Test language**: go
- **How to run**: Create the target test file with the body below, then run `go test ./cmd/... -run TestWriteParquetIgnoresFinalizationErrors -v`.

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

func TestWriteParquetIgnoresFinalizationErrors(t *testing.T) {
	t.Run("WriteStop returns a real finalization error", func(t *testing.T) {
		tmpDir := t.TempDir()
		path := filepath.Join(tmpDir, "broken.parquet")

		parquetFile, err := local.NewLocalFileWriter(path)
		if err != nil {
			t.Fatalf("NewLocalFileWriter failed: %v", err)
		}

		parquetWriter, err := writer.NewParquetWriter(parquetFile, new(transform.AssetOutputParquet), 1)
		if err != nil {
			t.Fatalf("NewParquetWriter failed: %v", err)
		}

		record := transform.AssetOutputParquet{
			AssetCode:   "XLM",
			AssetIssuer: "GABC",
			AssetType:   "native",
			AssetID:     123,
		}
		if err := parquetWriter.Write(record); err != nil {
			t.Fatalf("Write failed: %v", err)
		}

		if err := parquetFile.Close(); err != nil {
			t.Fatalf("Close failed: %v", err)
		}

		err = parquetWriter.WriteStop()
		if err == nil {
			t.Fatal("expected WriteStop to return an error after the file is closed")
		}

		stat, err := os.Stat(path)
		if err != nil {
			t.Fatalf("Stat failed: %v", err)
		}
		if stat.Size() >= 12 {
			t.Fatalf("expected a truncated parquet artifact, got %d bytes", stat.Size())
		}
	})

	t.Run("WriteParquet cannot propagate those errors", func(t *testing.T) {
		_, thisFile, _, ok := runtime.Caller(0)
		if !ok {
			t.Fatal("runtime.Caller failed")
		}

		sourcePath := filepath.Join(filepath.Dir(thisFile), "command_utils.go")
		fileSet := token.NewFileSet()
		parsed, err := parser.ParseFile(fileSet, sourcePath, nil, 0)
		if err != nil {
			t.Fatalf("ParseFile failed: %v", err)
		}

		foundWriteParquet := false
		foundDeferredWriteStop := false
		foundDeferredClose := false

		for _, decl := range parsed.Decls {
			fn, ok := decl.(*ast.FuncDecl)
			if !ok || fn.Name.Name != "WriteParquet" {
				continue
			}
			foundWriteParquet = true

			if fn.Type.Results != nil && len(fn.Type.Results.List) > 0 {
				t.Fatal("WriteParquet now returns a value; hypothesis is outdated")
			}

			ast.Inspect(fn.Body, func(node ast.Node) bool {
				deferStmt, ok := node.(*ast.DeferStmt)
				if !ok {
					return true
				}

				selector, ok := deferStmt.Call.Fun.(*ast.SelectorExpr)
				if !ok {
					return true
				}

				switch selector.Sel.Name {
				case "WriteStop":
					foundDeferredWriteStop = true
				case "Close":
					foundDeferredClose = true
				}

				return true
			})
		}

		if !foundWriteParquet {
			t.Fatal("WriteParquet not found")
		}
		if !foundDeferredWriteStop {
			t.Fatal("WriteParquet no longer defers WriteStop")
		}
		if !foundDeferredClose {
			t.Fatal("WriteParquet no longer defers Close")
		}
	})

	t.Run("successful WriteParquet writes a materially larger file", func(t *testing.T) {
		tmpDir := t.TempDir()
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

		goodBytes, err := os.ReadFile(goodPath)
		if err != nil {
			t.Fatalf("ReadFile failed: %v", err)
		}
		if len(goodBytes) < 12 {
			t.Fatalf("expected a complete parquet file, got %d bytes", len(goodBytes))
		}
		if string(goodBytes[len(goodBytes)-4:]) != "PAR1" {
			t.Fatal("expected a valid parquet footer")
		}
	})
}
```

## Expected vs Actual Behavior

- **Expected**: if parquet finalization fails, the export should surface the error and stop before treating the parquet artifact as complete.
- **Actual**: `WriteStop()`/`Close()` failures are discarded, so callers can continue after a truncated parquet write.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC proves the exact parquet writer stack returns a real finalization error and separately verifies the production helper has no mechanism to report it.
2. Realistic preconditions: YES — late filesystem errors on footer/index/trailer writes are ordinary I/O failures on full or quota-limited storage, and `WriteStop()` performs those writes after row acceptance.
3. Bug vs by-design: BUG — silent loss of finalization errors conflicts with the helper’s own fatal-on-write behavior and leaves callers believing the export completed.
4. Final severity: Medium — this is silent export failure/corruption under a specific I/O condition, not direct numeric field corruption.
5. In scope: YES — the result is a concrete data-export correctness issue that can leave downstream parquet consumers with corrupt or missing data.
6. Test correctness: CORRECT — the PoC does not rely on tautological assertions; it demonstrates a real `WriteStop()` error, a truncated artifact, and the production helper’s lack of error propagation.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Change `WriteParquet` to return an `error`, call `WriteStop()` and `Close()` explicitly, and propagate whichever finalization error occurs first back to the export command so it can abort before upload or success reporting.
