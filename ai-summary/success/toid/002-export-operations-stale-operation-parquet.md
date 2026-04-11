# 002: `export_operations` uploads stale parquet before regenerating current TOIDs

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness / stale cloud parquet upload
**Subsystem**: toid
**Final review by**: gpt-5.4, high

## Summary

`export_operations` calls `MaybeUpload(parquetPath)` before `WriteParquet(...)`, so cloud upload happens against whatever file already exists at that path instead of the current run's freshly generated operation parquet. On a first run this crashes when GCS tries to open a missing parquet file; on repeated runs it silently uploads the previous run's `transaction_id` and `id` TOIDs while the local file is only regenerated afterward.

The original PoC needed correction because its target test file did not exist and its stale-upload proof was too synthetic. After rewriting the PoC to use the production `WriteParquet()` path and the real `GCS.UploadTo()` missing-file behavior, the bug is reproducible and the source trace leaves no benign explanation.

## Root Cause

The parquet block in `export_operations` is ordered as upload-then-write instead of write-then-upload. `MaybeUpload()` immediately dispatches to `GCS.UploadTo()`, which opens the path that exists on disk at call time, while `WriteParquet()` is the first function that actually recreates the current run's parquet artifact.

## Reproduction

During normal operation, `export_operations --write-parquet --cloud-provider gcp` uses the shared archive default `exported_operations.parquet` unless the operator overrides `--parquet-output`. That means repeated exports naturally reuse the same local parquet path; the command uploads that old file first, then overwrites it locally with new TOIDs for the current ledger range.

## Affected Code

- `internal/utils/main.go:AddArchiveFlags:250-253` — archive exporters default to a reusable `exported_<object>.parquet` path
- `cmd/export_operations.go:63-65` — uploads `parquetPath` before regenerating it
- `cmd/command_utils.go:123-146` — `MaybeUpload()` immediately invokes the cloud uploader and fatals on upload errors
- `cmd/upload_to_gcs.go:25-35` — `UploadTo()` opens the on-disk path before any upload work
- `cmd/command_utils.go:162-180` — `WriteParquet()` is the first point that creates the current parquet file contents

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportParquetUploadOrderingBug`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go build ./... && go test ./cmd -run TestExportParquetUploadOrderingBug -v`.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"runtime"
	"strings"
	"testing"
	"time"

	"github.com/stellar/stellar-etl/v2/internal/toid"
	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/xitongsys/parquet-go-source/local"
	parquetreader "github.com/xitongsys/parquet-go/reader"
)

func TestExportParquetUploadOrderingBug(t *testing.T) {
	t.Run("source_ordering", func(t *testing.T) {
		source := readCmdSource(t, "export_operations.go")

		maybeUploadIndex := strings.Index(source, "MaybeUpload(cloudCredentials, cloudStorageBucket, cloudProvider, parquetPath)")
		writeParquetIndex := strings.Index(source, "WriteParquet(transformedOps, parquetPath, new(transform.OperationOutputParquet))")
		if maybeUploadIndex == -1 || writeParquetIndex == -1 {
			t.Fatalf("could not locate parquet upload/write calls in export_operations.go")
		}

		if maybeUploadIndex > writeParquetIndex {
			t.Fatalf("WriteParquet already runs before MaybeUpload; the ordering bug appears fixed")
		}
	})

	t.Run("first_run_missing_parquet_fails_in_production_upload_path", func(t *testing.T) {
		missingPath := filepath.Join(t.TempDir(), "operations.parquet")

		err := (&GCS{}).UploadTo("", "unused-bucket", missingPath)
		if err == nil {
			t.Fatal("expected UploadTo to fail for a missing parquet path")
		}
		if !strings.Contains(err.Error(), "failed to open file") {
			t.Fatalf("expected missing-file failure from UploadTo, got %v", err)
		}
	})

	t.Run("reused_path_uploads_previous_run_operation_ids", func(t *testing.T) {
		parquetPath := filepath.Join(t.TempDir(), "operations.parquet")

		staleTxID := toid.New(100, 1, 0).ToInt64()
		staleOpID := toid.New(100, 1, 1).ToInt64()
		freshTxID := toid.New(200, 1, 0).ToInt64()
		freshOpID := toid.New(200, 1, 1).ToInt64()

		WriteParquet([]transform.SchemaParquet{
			operationParquetFixture(staleTxID, staleOpID, 100),
		}, parquetPath, new(transform.OperationOutputParquet))

		uploadedRows := readOperationParquetRows(t, parquetPath)
		if len(uploadedRows) != 1 {
			t.Fatalf("expected 1 row before regeneration, got %d", len(uploadedRows))
		}
		if uploadedRows[0].TransactionID != staleTxID || uploadedRows[0].OperationID != staleOpID {
			t.Fatalf("expected stale parquet row to contain transaction_id=%d and id=%d, got transaction_id=%d id=%d",
				staleTxID, staleOpID, uploadedRows[0].TransactionID, uploadedRows[0].OperationID)
		}

		WriteParquet([]transform.SchemaParquet{
			operationParquetFixture(freshTxID, freshOpID, 200),
		}, parquetPath, new(transform.OperationOutputParquet))

		freshRows := readOperationParquetRows(t, parquetPath)
		if len(freshRows) != 1 {
			t.Fatalf("expected 1 row after regeneration, got %d", len(freshRows))
		}
		if freshRows[0].TransactionID != freshTxID || freshRows[0].OperationID != freshOpID {
			t.Fatalf("expected regenerated parquet row to contain transaction_id=%d and id=%d, got transaction_id=%d id=%d",
				freshTxID, freshOpID, freshRows[0].TransactionID, freshRows[0].OperationID)
		}
		if uploadedRows[0].TransactionID == freshRows[0].TransactionID || uploadedRows[0].OperationID == freshRows[0].OperationID {
			t.Fatalf("expected stale and regenerated parquet rows to differ, got before=%+v after=%+v", uploadedRows[0], freshRows[0])
		}
	})
}

func readCmdSource(t *testing.T, name string) string {
	t.Helper()

	_, thisFile, _, ok := runtime.Caller(0)
	if !ok {
		t.Fatal("could not determine current file location")
	}

	source, err := os.ReadFile(filepath.Join(filepath.Dir(thisFile), name))
	if err != nil {
		t.Fatalf("could not read %s: %v", name, err)
	}

	return string(source)
}

func operationParquetFixture(txID, opID int64, ledgerSequence uint32) transform.OperationOutput {
	return transform.OperationOutput{
		SourceAccount:       "GAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAWHF",
		Type:                1,
		TypeString:          "payment",
		OperationDetails:    map[string]interface{}{"amount": "1.0000000"},
		TransactionID:       txID,
		OperationID:         opID,
		ClosedAt:            time.Unix(int64(ledgerSequence), 0).UTC(),
		OperationResultCode: "opINNER",
		OperationTraceCode:  "",
		LedgerSequence:      ledgerSequence,
	}
}

func readOperationParquetRows(t *testing.T, path string) []transform.OperationOutputParquet {
	t.Helper()

	readerFile, err := local.NewLocalFileReader(path)
	if err != nil {
		t.Fatalf("could not open parquet file %s: %v", path, err)
	}
	defer readerFile.Close()

	reader, err := parquetreader.NewParquetReader(readerFile, new(transform.OperationOutputParquet), 1)
	if err != nil {
		t.Fatalf("could not create parquet reader: %v", err)
	}
	defer reader.ReadStop()

	rows := make([]transform.OperationOutputParquet, int(reader.GetNumRows()))
	if err := reader.Read(&rows); err != nil {
		t.Fatalf("could not read parquet rows: %v", err)
	}

	return rows
}
```

## Expected vs Actual Behavior

- **Expected**: `export_operations --write-parquet` should regenerate the current run's parquet file before any cloud upload, so uploaded `transaction_id` and `id` TOIDs match the current ledger range.
- **Actual**: the command uploads the existing parquet path first, so first runs fail on a missing file and repeated runs upload the previous run's TOIDs while only the local file is updated afterward.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC proves the exact `export_operations` call order, uses the real `GCS.UploadTo()` missing-file path, and uses the real `WriteParquet()` plus parquet reader path to show previous-run TOIDs differ from the current run's regenerated row.
2. Realistic preconditions: YES — cloud upload is a documented first-class mode, and repeated runs reuse the default `exported_operations.parquet` path unless explicitly overridden.
3. Bug vs by-design: BUG — sibling exporters such as `export_assets` use the correct write-then-upload order, and nothing in the command contract suggests uploads should target stale prior-run files.
4. Final severity: Medium — this is an output lifecycle bug that either fatals on first use or silently ships stale parquet data to cloud storage while reporting success.
5. In scope: YES — the finding is a concrete exporter correctness failure that produces wrong but plausible downstream data.
6. Test correctness: CORRECT — the rewritten PoC avoids circular assertions, checks the actual production helpers involved, and demonstrates a real difference in exported TOID-bearing parquet rows.
7. Alternative explanations: NONE — once `MaybeUpload(parquetPath)` is called before `WriteParquet(...)`, the uploaded artifact can only be missing or stale.
8. Novelty: NOVEL

## Suggested Fix

Swap the parquet block in `export_operations` to `WriteParquet(...)` first and `MaybeUpload(parquetPath)` second, matching the already-correct ordering used by `export_assets` and the other fixed exporters. The same ordering audit should be applied to sibling commands that still upload parquet before writing it.
