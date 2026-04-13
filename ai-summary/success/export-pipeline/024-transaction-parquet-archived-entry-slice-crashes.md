# 024: Transaction Parquet archived-entry slices break writer

**Date**: 2026-04-13
**Severity**: Medium
**Impact**: Data loss under specific conditions
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`export_transactions --write-parquet` breaks on valid Soroban transactions when `soroban_resources_archived_entries` is non-empty. The transaction transform emits that field as `[]uint32`, but the Parquet schema tags it as repeated `INT32`, so parquet-go errors out as soon as it marshals a real element.

## Root Cause

`TransformTransaction()` copies archived footprint entry indexes from Soroban XDR into `TransactionOutput.SorobanResourcesArchivedEntries` as `[]uint32`. `TransactionOutput.ToParquet()` forwards that slice unchanged into `TransactionOutputParquet`, whose field is still `[]uint32` but tagged as repeated `INT32`; parquet-go's `INT32` path calls `reflect.Value.Int()` on each element and rejects unsigned values.

## Reproduction

This occurs during normal `export_transactions --write-parquet` operation for Soroban transactions whose footprint extension contains archived-entry indexes. The reproduced PoC used the same `writer.NewParquetWriter(..., new(TransactionOutputParquet), 1)` plus `writer.Write(record.ToParquet())` path that `cmd.WriteParquet()` uses, and `writer.Write(...)` returned `reflect: call of reflect.Value.Int on uint32 Value` for a row with `SorobanResourcesArchivedEntries: []uint32{1, 2}` while an empty slice succeeded.

## Affected Code

- `internal/transform/transaction.go:171-174` — copies archived Soroban footprint indexes into `[]uint32`
- `internal/transform/schema.go:75` — exposes `soroban_resources_archived_entries` as `[]uint32` in the transaction output model
- `internal/transform/schema_parquet.go:66` — tags `SorobanResourcesArchivedEntries []uint32` as repeated `INT32`
- `internal/transform/parquet_converter.go:94` — forwards the `[]uint32` slice unchanged into the Parquet struct
- `cmd/command_utils.go:169-177` — `WriteParquet()` constructs the writer and fatals on the first `writer.Write(...)` error
- `cmd/export_transactions.go:63-66` — `export_transactions` always routes `--write-parquet` through `WriteParquet(..., new(transform.TransactionOutputParquet))`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTransactionParquetArchivedEntryCrash`
- **Test language**: `go`
- **How to run**: Append the function below to the target test file, ensure that file imports `os`, `github.com/xitongsys/parquet-go-source/local`, and `github.com/xitongsys/parquet-go/writer`, then run `go test ./internal/transform/... -run TestTransactionParquetArchivedEntryCrash -v`.

### Test Body

```go
func TestTransactionParquetArchivedEntryCrash(t *testing.T) {
	tmpFile, err := os.CreateTemp("", "poc-archived-*.parquet")
	if err != nil {
		t.Fatalf("failed to create temp file: %v", err)
	}
	tmpPath := tmpFile.Name()
	tmpFile.Close()
	defer os.Remove(tmpPath)

	pf, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatalf("failed to create parquet file writer: %v", err)
	}
	defer pf.Close()

	pw, err := writer.NewParquetWriter(pf, new(TransactionOutputParquet), 1)
	if err != nil {
		t.Fatalf("failed to create parquet writer: %v", err)
	}

	record := TransactionOutputParquet{
		SorobanResourcesArchivedEntries: []uint32{1, 2},
	}

	writeErr := pw.Write(record)
	stopErr := pw.WriteStop()

	if writeErr == nil && stopErr == nil {
		t.Fatal("expected an error from Write or WriteStop for non-empty []uint32 archived entries, but got nil")
	}

	combinedErr := ""
	if writeErr != nil {
		combinedErr += writeErr.Error()
	}
	if stopErr != nil {
		combinedErr += stopErr.Error()
	}
	t.Logf("Bug confirmed: non-empty []uint32 archived entries causes error: %s", combinedErr)

	tmpFile2, err := os.CreateTemp("", "poc-empty-*.parquet")
	if err != nil {
		t.Fatalf("failed to create second temp file: %v", err)
	}
	tmpPath2 := tmpFile2.Name()
	tmpFile2.Close()
	defer os.Remove(tmpPath2)

	pf2, err := local.NewLocalFileWriter(tmpPath2)
	if err != nil {
		t.Fatalf("failed to create second parquet file writer: %v", err)
	}
	defer pf2.Close()

	pw2, err := writer.NewParquetWriter(pf2, new(TransactionOutputParquet), 1)
	if err != nil {
		t.Fatalf("failed to create second parquet writer: %v", err)
	}

	emptyRecord := TransactionOutputParquet{
		SorobanResourcesArchivedEntries: []uint32{},
	}

	if err := pw2.Write(emptyRecord); err != nil {
		t.Fatalf("empty slice should not error on Write, got: %v", err)
	}
	if err := pw2.WriteStop(); err != nil {
		t.Fatalf("empty slice should not error on WriteStop, got: %v", err)
	}

	t.Log("Confirmed: empty []uint32 slice writes successfully, proving bug is triggered only by non-empty data")
}
```

## Expected vs Actual Behavior

- **Expected**: `export_transactions --write-parquet` should serialize non-empty archived-entry index lists for Soroban transactions and continue writing rows.
- **Actual**: the first non-empty archived-entry slice makes parquet-go return `reflect: call of reflect.Value.Int on uint32 Value`, and `WriteParquet()` aborts the export.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC hits the same `TransactionOutputParquet` writer path used by `record.ToParquet()` inside `cmd.WriteParquet()`.
2. Realistic preconditions: YES — upstream Stellar XDR defines archived Soroban footprint indexes as `[]Uint32`, and real Soroban transactions can populate that extension.
3. Bug vs by-design: BUG — the field is modeled as unsigned integers but tagged for Parquet's signed `INT32` conversion path, which is an incompatible encoding rather than an intentional export policy.
4. Final severity: Medium — the result is an operational export failure on the Parquet path, not silent monetary corruption of an emitted value.
5. In scope: YES — this is a concrete repository bug that breaks advertised transaction Parquet export for valid chain data.
6. Test correctness: CORRECT — it uses the production schema type and parquet-go writer with no mocks or tautological assertions, and it proves the empty-slice control case succeeds.
7. Alternative explanations: NONE — the failure disappears when the slice is empty and reappears when actual `uint32` elements are marshaled.
8. Novelty: NOVEL

## Suggested Fix

Convert `SorobanResourcesArchivedEntries` to a signed Parquet-compatible slice before writing (for example `[]int32` in the Parquet struct and converter), or retag and convert the field with a Parquet encoding that genuinely supports unsigned 32-bit repeated elements. Add a regression test that writes a transaction row with non-empty archived-entry indexes through the Parquet path.
