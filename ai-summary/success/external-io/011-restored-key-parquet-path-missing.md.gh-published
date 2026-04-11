# 011: Restored-key Parquet path missing

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes --export-restored-keys --write-parquet` writes real restored-key rows to JSON, then emits a Parquet file with zero rows for the same batch. Downstream Parquet consumers silently lose every restored key even though the export completed successfully and the JSON side proves the data existed.

## Root Cause

The restored-key export path appends `transform.RestoredKeyOutput` rows into `transformedOutputs["restored_key"]`, but `exportTransformedData()` has no `case transform.RestoredKeyOutput` in its Parquet type switch. That leaves `transformedResource` empty and `parquetSchema` nil, yet the code still calls `WriteParquet(transformedResource, parquetPath, parquetSchema)`, which produces a syntactically valid zero-row Parquet file instead of preserving the restored-key rows.

## Reproduction

Any normal `export_ledger_entry_changes --export-restored-keys --write-parquet` run that encounters restored ledger-entry changes will hit this path. The command transforms restored entries into JSON rows first, then the Parquet conversion step drops them because restored keys have no Parquet case or schema wiring.

## Affected Code

- `cmd/export_ledger_entry_changes.go:Run:90-125` - maps `--export-restored-keys` to `restored_key` output and appends real `RestoredKeyOutput` rows
- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-372` - omits `RestoredKeyOutput` from the Parquet switch and still writes Parquet for the resource
- `cmd/command_utils.go:WriteParquet:162-180` - accepts the empty slice/nil schema combination and emits an empty Parquet artifact

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestRestoredKeyParquetTypeSwitchGap`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"testing"
	"time"

	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/xitongsys/parquet-go/reader"
	"github.com/xitongsys/parquet-go-source/local"
)

const restoredKeyParquetHelperEnv = "STELLAR_ETL_RESTORED_KEY_PARQUET_HELPER"
const restoredKeyJSONDirEnv = "STELLAR_ETL_RESTORED_KEY_JSON_DIR"
const restoredKeyParquetDirEnv = "STELLAR_ETL_RESTORED_KEY_PARQUET_DIR"

func TestRestoredKeyParquetTypeSwitchGap(t *testing.T) {
	if os.Getenv(restoredKeyParquetHelperEnv) == "1" {
		runRestoredKeyParquetHelper()
		return
	}

	tempDir := t.TempDir()
	jsonDir := filepath.Join(tempDir, "json")
	parquetDir := filepath.Join(tempDir, "parquet")
	workingDir, err := os.Getwd()
	if err != nil {
		t.Fatalf("failed to get working directory: %v", err)
	}

	cmd := exec.Command(os.Args[0], "-test.run=^TestRestoredKeyParquetTypeSwitchGap$")
	cmd.Dir = filepath.Join(workingDir, "cmd")
	cmd.Env = append(os.Environ(),
		restoredKeyParquetHelperEnv+"=1",
		restoredKeyJSONDirEnv+"="+jsonDir,
		restoredKeyParquetDirEnv+"="+parquetDir,
	)

	output, err := cmd.CombinedOutput()
	if err != nil {
		t.Fatalf("expected restored-key parquet export subprocess to succeed, got %v with output:\n%s", err, string(output))
	}

	jsonPath := filepath.Join(jsonDir, exportFilename(58764192, 58764194, "restored_key"))
	jsonBytes, readErr := os.ReadFile(jsonPath)
	if readErr != nil {
		t.Fatalf("expected JSON export file to exist before parquet failure: %v", readErr)
	}
	if !strings.Contains(string(jsonBytes), "\"ledger_key_hash\":\"AAAAAA==\"") {
		t.Fatalf("expected JSON export to contain restored key row, got %s", string(jsonBytes))
	}

	parquetPath := filepath.Join(parquetDir, exportParquetFilename(58764192, 58764194, "restored_key"))
	info, statErr := os.Stat(parquetPath)
	if statErr != nil {
		t.Fatalf("expected parquet file to exist: %v", statErr)
	}
	if info.Size() == 0 {
		t.Fatal("expected parquet writer to emit a syntactically valid file")
	}

	rowCount, err := countParquetRows(parquetPath)
	if err != nil {
		t.Fatalf("failed to read restored-key parquet file: %v", err)
	}
	if rowCount != 0 {
		t.Fatalf("expected restored-key parquet file to contain zero rows, got %d", rowCount)
	}
}

func runRestoredKeyParquetHelper() {
	jsonDir := os.Getenv(restoredKeyJSONDirEnv)
	parquetDir := os.Getenv(restoredKeyParquetDirEnv)
	if err := os.MkdirAll(jsonDir, 0o755); err != nil {
		panic(err)
	}
	if err := os.MkdirAll(parquetDir, 0o755); err != nil {
		panic(err)
	}

	transformedOutput := map[string][]interface{}{
		"restored_key": {
			transform.RestoredKeyOutput{
				LedgerKeyHash:      "AAAAAA==",
				LedgerEntryType:    "LedgerEntryTypeOffer",
				LastModifiedLedger: 58764192,
				ClosedAt:           time.Date(2024, time.January, 1, 0, 0, 0, 0, time.UTC),
				LedgerSequence:     58764193,
			},
		},
	}

	_ = exportTransformedData(
		58764192,
		58764193,
		jsonDir,
		parquetDir,
		transformedOutput,
		"",
		"",
		"",
		nil,
		true,
	)
}

func countParquetRows(path string) (int64, error) {
	fileReader, err := local.NewLocalFileReader(path)
	if err != nil {
		return 0, err
	}
	defer fileReader.Close()

	parquetReader, err := reader.NewParquetReader(fileReader, nil, 1)
	if err != nil {
		return 0, err
	}
	defer parquetReader.ReadStop()

	return parquetReader.GetNumRows(), nil
}
```

## Expected vs Actual Behavior

- **Expected**: When restored-key JSON rows are exported for a batch, the matching Parquet file contains the same restored-key rows.
- **Actual**: The export completes successfully, JSON contains restored-key rows, and the generated Parquet file for `restored_key` contains zero rows.

## Adversarial Review

1. Exercises claimed bug: YES - the PoC calls the real `exportTransformedData()` path and inspects the actual on-disk JSON and Parquet outputs it generates.
2. Realistic preconditions: YES - restored-key rows already exist in normal ledger-entry change exports, and `--write-parquet` is a supported production flag.
3. Bug vs by-design: BUG - unlike explicitly skipped claimable balances, restored keys are exported to JSON and exposed via CLI flags, so emitting an empty Parquet file is an unintended omission.
4. Final severity: High - this is structural export corruption that silently removes non-financial rows from one output format.
5. In scope: YES - it is a concrete production export path producing wrong persisted output.
6. Test correctness: CORRECT - the test verifies the same batch yields populated JSON plus a zero-row Parquet file, rather than asserting on copied switch logic.
7. Alternative explanations: NONE
8. Novelty: NOT ASSESSED - duplicate handling is owned by the orchestrator, and the underlying export bug is confirmed regardless

## Suggested Fix

Add full Parquet support for `RestoredKeyOutput`: define a `RestoredKeyOutputParquet` schema, implement `RestoredKeyOutput.ToParquet()`, and add a `case transform.RestoredKeyOutput` branch in `exportTransformedData()`. If restored keys are intentionally unsupported in Parquet, then the exporter should explicitly `skip = true` and surface that limitation instead of writing a misleading empty file.
