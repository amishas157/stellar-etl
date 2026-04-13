# 023: Ledger Parquet evicted-key tags break writer

**Date**: 2026-04-13
**Severity**: Medium
**Impact**: Data loss under specific conditions
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`export_ledgers --write-parquet` cannot initialize its Parquet writer because `LedgerOutputParquet` models `evicted_ledger_keys_type` and `evicted_ledger_keys_hash` as `[]string` but tags them like scalar UTF-8 columns. `writer.NewParquetWriter` rejects that schema before the first row is written, so the ledger Parquet path aborts instead of producing output.

## Root Cause

`LedgerOutputParquet` omits the list element metadata that parquet-go requires for `[]string` fields. `LedgerOutput.ToParquet()` forwards the two slices unchanged, and `cmd.WriteParquet()` immediately fatals if `writer.NewParquetWriter(...)` cannot build the schema, so the invalid tags break the entire export path.

## Reproduction

This manifests during normal `export_ledgers --write-parquet` usage and does not depend on any special ledger contents. The reproduced PoC directly exercised the production writer-construction path with `new(LedgerOutputParquet)` and got `failed to create schema from tag map: type : not a valid Type string`, which is the same failure `cmd.WriteParquet()` turns into a fatal export abort.

## Affected Code

- `internal/transform/schema_parquet.go:27-28` — `EvictedLedgerKeysType` and `EvictedLedgerKeysHash` are `[]string` with scalar UTF-8 tags instead of valid list/repeated tags
- `internal/transform/parquet_converter.go:27-56` — `LedgerOutput.ToParquet()` passes both slices directly into `LedgerOutputParquet`
- `cmd/command_utils.go:162-172` — `WriteParquet()` constructs the Parquet writer and fatals on schema-init failure
- `cmd/export_ledgers.go:70-72` — `export_ledgers` always uses `LedgerOutputParquet` when `--write-parquet` is set

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestLedgerParquetEvictedKeyTagsBreakWriter`
- **Test language**: `go`
- **How to run**: Append the function below to the target test file, ensure that file imports `os`, `strings`, `github.com/xitongsys/parquet-go-source/local`, and `github.com/xitongsys/parquet-go/writer`, then run `go test ./internal/transform/... -run TestLedgerParquetEvictedKeyTagsBreakWriter -v`.

### Test Body

```go
func TestLedgerParquetEvictedKeyTagsBreakWriter(t *testing.T) {
	tmpFile, err := os.CreateTemp("", "ledger_parquet_poc_*.parquet")
	if err != nil {
		t.Fatalf("failed to create temp file: %v", err)
	}
	tmpPath := tmpFile.Name()
	tmpFile.Close()
	defer os.Remove(tmpPath)

	pf, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatalf("failed to create local file writer: %v", err)
	}
	defer pf.Close()

	pw, err := writer.NewParquetWriter(pf, new(LedgerOutputParquet), 1)
	if pw != nil {
		pw.WriteStop()
	}

	if err == nil {
		t.Fatal("expected NewParquetWriter to fail for LedgerOutputParquet, but it succeeded")
	}

	if !strings.Contains(err.Error(), "not a valid Type string") {
		t.Errorf("unexpected error message: got %q, want substring %q", err.Error(), "not a valid Type string")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_ledgers --write-parquet` should initialize a Parquet writer and emit ledger rows, regardless of whether the evicted-key arrays are empty or populated.
- **Actual**: Parquet writer creation fails immediately on the invalid `[]string` schema tags, so the command exits before writing any Parquet rows.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the exact `writer.NewParquetWriter(..., new(LedgerOutputParquet), 1)` path that `cmd.WriteParquet()` uses.
2. Realistic preconditions: YES — the only requirement is normal use of `export_ledgers --write-parquet`.
3. Bug vs by-design: BUG — sibling slice fields in this codebase use valid LIST or REPEATED parquet tags, while these two tags are inconsistent and break advertised Parquet support.
4. Final severity: Medium — the export fails loudly and writes no Parquet rows, causing operational data loss on the Parquet path rather than silent field corruption.
5. In scope: YES — this is a concrete correctness failure in repository code.
6. Test correctness: CORRECT — the assertion is on the real schema-init error from the production schema type, with no mocks or circular setup.
7. Alternative explanations: NONE — the failure happens during schema construction, before any row data or downstream upload logic can influence the result.
8. Novelty: NOVEL

## Suggested Fix

Retag both evicted-key fields with a valid Parquet list encoding, following the working slice patterns already used elsewhere in `schema_parquet.go`, and add a regression test that `writer.NewParquetWriter` succeeds for `LedgerOutputParquet`.
