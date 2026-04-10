# 003: Restored-key parquet export silently drops all rows

**Date**: 2026-04-10
**Severity**: High
**Impact**: structural data corruption
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes` supports `--export-restored-keys`, and the JSON export path emits correct `restored_key` rows. When the same command is run with `--write-parquet`, the command still creates a `restored_key` parquet file but serializes zero rows into it, silently diverging from the JSON export for the same batch.

## Root Cause

`exportTransformedData` appends `transform.RestoredKeyOutput` into `transformedOutputs["restored_key"]`, but its parquet type switch has no `transform.RestoredKeyOutput` case. That leaves both `transformedResource` and `parquetSchema` unset, yet the function still calls `WriteParquet`, which writes a parquet file with zero rows instead of the restored-key data.

## Reproduction

During normal operation, a user can combine the documented generic `--write-parquet` flag with the documented `--export-restored-keys` flag on `export_ledger_entry_changes`. Any ledger range containing `LedgerEntryRestored` changes will then produce matching JSON data and an empty restored-key parquet artifact for the same batch.

## Affected Code

- `cmd/export_ledger_entry_changes.go:Run:111-126` — collects restored-key rows into `transformedOutputs["restored_key"]`
- `cmd/export_ledger_entry_changes.go:exportTransformedData:304-372` — omits `transform.RestoredKeyOutput` from the parquet switch but still calls `WriteParquet`
- `cmd/command_utils.go:WriteParquet:162-180` — writes whatever schema/data pair it is given, including the nil/nil restored-key case

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestRestoredKeyParquetPathMissing`
- **Test language**: go
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"encoding/json"
	"os"
	"path/filepath"
	"testing"
	"time"

	"github.com/stellar/stellar-etl/v2/internal/transform"
	parquetlocal "github.com/xitongsys/parquet-go-source/local"
	parquetreader "github.com/xitongsys/parquet-go/reader"
)

func TestRestoredKeyParquetPathMissing(t *testing.T) {
	tmpDir := t.TempDir()
	outputFolder := filepath.Join(tmpDir, "json")
	parquetFolder := filepath.Join(tmpDir, "parquet")
	if err := os.MkdirAll(outputFolder, 0o755); err != nil {
		t.Fatalf("creating json output dir: %v", err)
	}
	if err := os.MkdirAll(parquetFolder, 0o755); err != nil {
		t.Fatalf("creating parquet output dir: %v", err)
	}

	expected := transform.RestoredKeyOutput{
		LedgerKeyHash:      "abc123",
		LedgerEntryType:    "contract_data",
		LastModifiedLedger: 100,
		ClosedAt:           time.Unix(1_710_000_000, 0).UTC(),
		LedgerSequence:     100,
	}

	transformedOutputs := map[string][]interface{}{
		"restored_key": {expected},
	}

	if err := exportTransformedData(0, 1, outputFolder, parquetFolder, transformedOutputs, "", "", "", nil, true); err != nil {
		t.Fatalf("exportTransformedData returned error: %v", err)
	}

	jsonPath := filepath.Join(outputFolder, exportFilename(0, 2, "restored_key"))
	jsonData, err := os.ReadFile(jsonPath)
	if err != nil {
		t.Fatalf("reading json output: %v", err)
	}

	var actualJSON transform.RestoredKeyOutput
	if err := json.Unmarshal(jsonData, &actualJSON); err != nil {
		t.Fatalf("unmarshalling json output: %v", err)
	}
	if actualJSON != expected {
		t.Fatalf("json output mismatch: got %+v want %+v", actualJSON, expected)
	}

	parquetPath := filepath.Join(parquetFolder, exportParquetFilename(0, 2, "restored_key"))
	parquetFile, err := parquetlocal.NewLocalFileReader(parquetPath)
	if err != nil {
		t.Fatalf("opening parquet output: %v", err)
	}
	defer parquetFile.Close()

	parquetReader, err := parquetreader.NewParquetColumnReader(parquetFile, 1)
	if err != nil {
		t.Fatalf("creating parquet reader: %v", err)
	}
	defer parquetReader.ReadStop()

	if gotRows := parquetReader.GetNumRows(); gotRows != 0 {
		t.Fatalf("expected restored_key parquet export to contain 0 rows due to missing parquet mapping, got %d", gotRows)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: The restored-key parquet export should contain the same single row that the JSON export writes for the batch.
- **Actual**: The JSON export contains the restored-key row, but the parquet artifact is created with zero rows.

## Adversarial Review

1. Exercises claimed bug: YES — the test calls the production `exportTransformedData` path with a real `transform.RestoredKeyOutput` value and `writeParquet=true`.
2. Realistic preconditions: YES — both `--export-restored-keys` and the generic `--write-parquet` flag are public CLI flags, and restored ledger-entry changes occur on-chain.
3. Bug vs by-design: BUG — unsupported parquet types are explicitly marked with `skip = true` for claimable balances; restored keys have neither a parquet mapping nor a skip guard.
4. Final severity: High — the command emits structurally wrong data for one advertised export type without surfacing an error.
5. In scope: YES — this is silent exported-data corruption in the CLI export pipeline.
6. Test correctness: CORRECT — the test validates the JSON row content and then reads parquet metadata to prove the generated parquet file contains zero rows.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Either add a full restored-key parquet implementation (`RestoredKeyOutputParquet`, `ToParquet()`, and a switch case in `exportTransformedData`) or explicitly set `skip = true` for `transform.RestoredKeyOutput` so the command does not emit a misleading empty parquet artifact.
