# 008: Parquet finalization errors are swallowed

**Date**: 2026-04-10
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`WriteParquet()` treats file creation, writer creation, and per-record writes as fatal, but it silently drops errors from the two finalization steps that actually finish the file: `writer.WriteStop()` and `parquetFile.Close()`. When finalization fails after rows have been accepted, the function returns normally and callers can continue with a corrupt or unreadable `.parquet` artifact.

## Root Cause

`cmd.WriteParquet()` uses bare deferred calls for `writer.WriteStop()` and `parquetFile.Close()`, so their `error` results are ignored. `parquet-go` performs footer/index writes inside `WriteStop()`, and the local parquet file wrapper returns `os.File.Close()` errors directly, so the code drops the last I/O failure surface of parquet generation.

## Reproduction

During normal operation, this manifests when the output path starts failing only at finalization time: delayed writeback failure, disk full, or a flaky mounted filesystem after row writes have already succeeded. `WriteParquet()` returns as if the export succeeded, and callers such as `export_contract_events` and `export_ledger_entry_changes` can then proceed with the unreadable parquet path.

## Affected Code

- `cmd/command_utils.go:WriteParquet:162-180` — defers `writer.WriteStop()` and `parquetFile.Close()` without checking their returned errors
- `cmd/export_contract_events.go:contractEventsCmd.Run:63-65` — calls `WriteParquet(...)` and then uploads the parquet path on apparent success
- `cmd/export_ledger_entry_changes.go:exportTransformedData:370-372` — same success-shaped follow-on path for batched parquet exports

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestWriteParquetFooterErrorSwallowed`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"testing"

	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/reader"
	"github.com/xitongsys/parquet-go/writer"
)

type pocParquetRow struct {
	ID   int64  `parquet:"name=id, type=INT64"`
	Name string `parquet:"name=name, type=BYTE_ARRAY, convertedtype=UTF8, encoding=PLAIN_DICTIONARY"`
}

func TestWriteParquetFooterErrorSwallowed(t *testing.T) {
	schema := new(pocParquetRow)

	baselinePath := t.TempDir() + "/baseline.parquet"

	pf, err := local.NewLocalFileWriter(baselinePath)
	if err != nil {
		t.Fatalf("baseline: cannot create parquet file: %v", err)
	}
	pw, err := writer.NewParquetWriter(pf, schema, 1)
	if err != nil {
		t.Fatalf("baseline: cannot create parquet writer: %v", err)
	}
	if err := pw.Write(pocParquetRow{ID: 1, Name: "test"}); err != nil {
		t.Fatalf("baseline: cannot write record: %v", err)
	}
	if err := pw.WriteStop(); err != nil {
		t.Fatalf("baseline: WriteStop failed: %v", err)
	}
	if err := pf.Close(); err != nil {
		t.Fatalf("baseline: close failed: %v", err)
	}

	baseReader, err := local.NewLocalFileReader(baselinePath)
	if err != nil {
		t.Fatalf("baseline: cannot open parquet file for reading: %v", err)
	}
	pr, err := reader.NewParquetReader(baseReader, schema, 1)
	if err != nil {
		t.Fatalf("baseline: cannot create parquet reader: %v", err)
	}
	if pr.GetNumRows() != 1 {
		t.Fatalf("baseline: expected 1 row, got %d", pr.GetNumRows())
	}
	pr.ReadStop()
	if err := baseReader.Close(); err != nil {
		t.Fatalf("baseline: reader close failed: %v", err)
	}

	corruptPath := t.TempDir() + "/corrupt.parquet"

	parquetFile, err := local.NewLocalFileWriter(corruptPath)
	if err != nil {
		t.Fatalf("could not create parquet file: %v", err)
	}

	pw2, err := writer.NewParquetWriter(parquetFile, schema, 1)
	if err != nil {
		t.Fatalf("could not create parquet writer: %v", err)
	}

	if err := pw2.Write(pocParquetRow{ID: 1, Name: "test"}); err != nil {
		t.Fatalf("could not write record: %v", err)
	}

	localFile := parquetFile.(*local.LocalFile)
	if err := localFile.File.Close(); err != nil {
		t.Fatalf("closing underlying file failed: %v", err)
	}

	writeStopErr := pw2.WriteStop()
	if writeStopErr == nil {
		t.Fatal("expected WriteStop() to return an error after fd was closed, but got nil")
	}

	corruptReader, err := local.NewLocalFileReader(corruptPath)
	if err != nil {
		t.Logf("corrupt file cannot be opened: %v", err)
		return
	}

	_, readerErr := reader.NewParquetReader(corruptReader, schema, 1)
	if err := corruptReader.Close(); err != nil {
		t.Fatalf("corrupt reader close failed: %v", err)
	}

	if readerErr == nil {
		t.Fatal("expected corrupt parquet file to be unreadable, but reader succeeded")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: If parquet finalization fails, the export should stop with an error before callers keep or upload the parquet path.
- **Actual**: `WriteParquet()` ignores `WriteStop()` and `Close()` failures, returns normally, and callers treat the parquet artifact as successfully generated.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses the same parquet writer stack as production, forces a non-nil `WriteStop()` result, and confirms the resulting file is unreadable.
2. Realistic preconditions: YES — the direct fd close is a deterministic fault-injection stand-in for real finalization failures such as disk-full or mounted-filesystem writeback errors.
3. Bug vs by-design: BUG — the function documentation says parquet write failures are fatal, and every earlier I/O failure in the function is already handled that way.
4. Final severity: Medium — this is silent export failure/data loss under specific I/O conditions, not direct financial field remapping.
5. In scope: YES — it is a concrete export-pipeline correctness bug that can leave downstream systems with an unreadable parquet object while the exporter reports success.
6. Test correctness: CORRECT — although the fault is injected synthetically, the test proves the real finalization path returns an error on the production writer stack and that the resulting artifact is invalid.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Replace the bare deferred calls with checked finalization, e.g. `if err := writer.WriteStop(); err != nil { cmdLogger.Fatal(...) }` and `if err := parquetFile.Close(); err != nil { cmdLogger.Fatal(...) }`, and only let callers proceed after both succeed.
