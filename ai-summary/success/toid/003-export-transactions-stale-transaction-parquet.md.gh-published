# 003: `export_transactions` uploads stale parquet before regenerating current transaction TOIDs

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness / stale cloud parquet upload
**Subsystem**: toid
**Final review by**: gpt-5.4, high

## Summary

`export_transactions` calls `MaybeUpload(parquetPath)` before `WriteParquet(...)`, so cloud upload runs against whatever file already exists at that path instead of the current run's freshly generated transaction parquet. On a first cloud-upload run this fails immediately because the parquet file does not exist yet; if a prior local parquet artifact is present, the command silently uploads that stale `id` value and only rewrites the local file afterward.

The original PoC overstated one trigger: a successful GCS upload deletes the local file, so "run it twice with cloud upload enabled" is not enough by itself to preserve stale parquet between runs. After correcting the PoC to cover both the real first-run failure path and the reused-path stale-data path, the finding remains valid.

## Root Cause

The parquet branch in `export_transactions` is ordered as upload-then-write instead of write-then-upload. `MaybeUpload()` immediately invokes `GCS.UploadTo()`, which opens the on-disk path at call time, while `WriteParquet()` is the first function that materializes the current run's parquet artifact.

## Reproduction

During normal operation, `export_transactions --write-parquet` uses the shared archive default `exported_transactions.parquet` unless the operator overrides `--parquet-output`. If cloud upload is enabled before that file exists locally, the command fatals on the missing parquet path; if a prior local parquet file is still present (for example from a previous non-upload run or failed upload attempt), the command uploads that older transaction parquet before regenerating the current ledger range locally.

## Affected Code

- `internal/utils/main.go:AddArchiveFlags:250-253` — archive exporters default to a reusable `exported_<object>.parquet` path
- `cmd/export_transactions.go:63-65` — uploads `parquetPath` before regenerating it
- `cmd/command_utils.go:123-146` — `MaybeUpload()` immediately invokes the cloud uploader and fatals on upload errors
- `cmd/upload_to_gcs.go:25-35` — `UploadTo()` opens the on-disk path before any upload work
- `cmd/command_utils.go:162-180` — `WriteParquet()` is the first point that creates the current parquet contents

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportTransactionsUploadsStaleParquet`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go build ./... && go test ./cmd/... -run TestExportTransactionsUploadsStaleParquet -v`.

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

func TestExportTransactionsUploadsStaleParquet(t *testing.T) {
	t.Run("source_ordering", func(t *testing.T) {
		source := readCmdSource(t, "export_transactions.go")

		maybeUploadIndex := strings.Index(source, "MaybeUpload(cloudCredentials, cloudStorageBucket, cloudProvider, parquetPath)")
		writeParquetIndex := strings.Index(source, "WriteParquet(transformedTransaction, parquetPath, new(transform.TransactionOutputParquet))")
		if maybeUploadIndex == -1 || writeParquetIndex == -1 {
			t.Fatal("could not locate parquet upload/write calls in export_transactions.go")
		}

		if maybeUploadIndex > writeParquetIndex {
			t.Fatal("WriteParquet already runs before MaybeUpload; the ordering bug appears fixed")
		}
	})

	t.Run("first_run_missing_parquet_fails_in_production_upload_path", func(t *testing.T) {
		missingPath := filepath.Join(t.TempDir(), "transactions.parquet")

		err := (&GCS{}).UploadTo("", "unused-bucket", missingPath)
		if err == nil {
			t.Fatal("expected UploadTo to fail for a missing parquet path")
		}
		if !strings.Contains(err.Error(), "failed to open file") {
			t.Fatalf("expected missing-file failure from UploadTo, got %v", err)
		}
	})

	t.Run("reused_path_exposes_previous_run_transaction_ids", func(t *testing.T) {
		parquetPath := filepath.Join(t.TempDir(), "transactions.parquet")

		staleTxID := toid.New(100, 1, 0).ToInt64()
		freshTxID := toid.New(200, 1, 0).ToInt64()

		WriteParquet([]transform.SchemaParquet{
			transactionParquetFixture(staleTxID, 100, "stale-run"),
		}, parquetPath, new(transform.TransactionOutputParquet))

		uploadedRows := readTransactionParquetRows(t, parquetPath)
		if len(uploadedRows) != 1 {
			t.Fatalf("expected 1 stale row before regeneration, got %d", len(uploadedRows))
		}
		if uploadedRows[0].TransactionID != staleTxID {
			t.Fatalf("expected stale parquet row to contain transaction_id=%d, got %d", staleTxID, uploadedRows[0].TransactionID)
		}

		WriteParquet([]transform.SchemaParquet{
			transactionParquetFixture(freshTxID, 200, "fresh-run"),
		}, parquetPath, new(transform.TransactionOutputParquet))

		freshRows := readTransactionParquetRows(t, parquetPath)
		if len(freshRows) != 1 {
			t.Fatalf("expected 1 row after regeneration, got %d", len(freshRows))
		}
		if freshRows[0].TransactionID != freshTxID {
			t.Fatalf("expected fresh parquet row to contain transaction_id=%d, got %d", freshTxID, freshRows[0].TransactionID)
		}
		if uploadedRows[0].TransactionID == freshRows[0].TransactionID {
			t.Fatalf("expected stale and regenerated transaction ids to differ, got before=%d after=%d", uploadedRows[0].TransactionID, freshRows[0].TransactionID)
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

func transactionParquetFixture(txID int64, ledgerSequence uint32, hash string) transform.TransactionOutput {
	return transform.TransactionOutput{
		TransactionHash: hash,
		LedgerSequence:  ledgerSequence,
		Account:         "GAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAWHF",
		AccountSequence: 1,
		MaxFee:          100,
		FeeCharged:      100,
		OperationCount:  1,
		TxEnvelope:      "envelope",
		TxResult:        "result",
		TxMeta:          "meta",
		TxFeeMeta:       "fee_meta",
		CreatedAt:       time.Unix(int64(ledgerSequence), 0).UTC(),
		MemoType:        "none",
		Successful:      true,
		TransactionID:   txID,
		ClosedAt:        time.Unix(int64(ledgerSequence), 0).UTC(),
	}
}

func readTransactionParquetRows(t *testing.T, path string) []transform.TransactionOutputParquet {
	t.Helper()

	readerFile, err := local.NewLocalFileReader(path)
	if err != nil {
		t.Fatalf("could not open parquet file %s: %v", path, err)
	}
	defer readerFile.Close()

	reader, err := parquetreader.NewParquetReader(readerFile, new(transform.TransactionOutputParquet), 1)
	if err != nil {
		t.Fatalf("could not create parquet reader: %v", err)
	}
	defer reader.ReadStop()

	rows := make([]transform.TransactionOutputParquet, int(reader.GetNumRows()))
	if err := reader.Read(&rows); err != nil {
		t.Fatalf("could not read parquet rows: %v", err)
	}

	return rows
}
```

## Expected vs Actual Behavior

- **Expected**: `export_transactions --write-parquet` should regenerate the current run's parquet file before any cloud upload, so uploaded transaction `id` values match the requested ledger range.
- **Actual**: the command uploads the existing parquet path first, so first cloud-upload runs fail on a missing file and reused paths expose prior-run transaction IDs while only the local parquet is updated afterward.

## Adversarial Review

1. Exercises claimed bug: YES — the corrected PoC proves the exact `export_transactions` call order, uses the real `GCS.UploadTo()` missing-file path, and uses the real `WriteParquet()` plus parquet reader path to show previous-run transaction IDs differ from the regenerated row.
2. Realistic preconditions: YES — cloud upload is a documented first-class mode, and archive exporters reuse a stable default parquet path; a successful GCS upload deletes the local file, so the stale-file case specifically requires a preserved local artifact from an earlier non-upload run or failed attempt.
3. Bug vs by-design: BUG — sibling exporters such as `export_assets` and `export_effects` use the correct write-then-upload order, and nothing in the command contract suggests uploads should target missing or stale prior-run parquet files.
4. Final severity: Medium — this is an output lifecycle bug that either fatals on first cloud use or silently ships stale parquet data while reporting success.
5. In scope: YES — the finding is a concrete exporter correctness failure that produces wrong but plausible downstream data.
6. Test correctness: CORRECT — the rewritten PoC avoids tautological assertions, checks the production helpers involved, and demonstrates a real difference in TOID-bearing parquet output.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Swap the parquet block in `export_transactions` to `WriteParquet(...)` first and `MaybeUpload(parquetPath)` second, matching the already-correct ordering used by `export_assets`, `export_effects`, and the other fixed exporters. The same ordering audit should remain applied to sibling commands that still upload parquet before writing it.
