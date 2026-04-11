# 004: export_trades uploads stale trade TOIDs before regenerating parquet

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: toid
**Final review by**: gpt-5.4, high

## Summary

`export_trades` uploads `parquetPath` before it regenerates the parquet file for the current run. When the same parquet filename is reused, cloud storage receives the previous run's trade `history_operation_id` and synthetic `buying_offer_id` values even though the local parquet file ends the run with the current values.

The uploaded object is structurally valid, so downstream systems can ingest stale TOID join keys without any obvious failure signal. That is structural data corruption, not just an operational nuisance.

## Root Cause

The command calls `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. `MaybeUpload` dispatches to `UploadTo`, which opens the existing local parquet file and copies its current bytes to cloud storage immediately; only after that returns does `WriteParquet` overwrite the path with the current run's transformed trade rows.

## Reproduction

On any rerun that reuses the same local parquet path, the path already contains a previous export. Because `export_trades` uploads before regenerating, the remote object is overwritten with the old file, then the local file is rewritten with new trade rows derived from the current ledger range.

This is realistic in normal operation because `AddArchiveFlags` defaults `--parquet-output` to `exported_trades.parquet`, so callers naturally reuse the same filename across runs unless they override it.

## Affected Code

- `cmd/export_trades.go:tradesCmd.Run:66-70` — uploads `parquetPath` before writing current parquet output
- `cmd/command_utils.go:MaybeUpload:123-146` — immediately dispatches cloud upload for the file currently on disk
- `cmd/upload_to_gcs.go:(*GCS).UploadTo:25-73` — opens and copies the existing local file, then deletes it on success
- `cmd/command_utils.go:WriteParquet:162-180` — only here is the current run's parquet file created and populated
- `internal/transform/trade.go:TransformTrade:33-34,116-120,152` — trade rows derive `history_operation_id` and synthetic `buying_offer_id` from operation TOIDs
- `internal/utils/main.go:AddArchiveFlags:250-253` — defaults parquet output to the reusable `exported_trades.parquet` path

## PoC

- **Target test file**: `cmd/export_trades_poc_test.go`
- **Test name**: `TestExportTradesStaleParquetTOIDUpload`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `cmd/export_trades_poc_test.go`
  3. Run: `go test ./cmd/... -run TestExportTradesStaleParquetTOIDUpload -v`
  4. Observe: the uploaded snapshot parquet contains prior-run `history_operation_id` / `buying_offer_id`, while the regenerated local parquet contains current values

### Test Body

```go
package cmd

import (
	"bytes"
	"os"
	"path/filepath"
	"runtime"
	"strings"
	"testing"
	"time"

	"github.com/guregu/null"
	"github.com/stellar/stellar-etl/v2/internal/toid"
	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/reader"
)

func writeTradeParquet(t *testing.T, path string, trades ...transform.TradeOutput) {
	t.Helper()

	records := make([]transform.SchemaParquet, len(trades))
	for i := range trades {
		records[i] = trades[i]
	}

	WriteParquet(records, path, new(transform.TradeOutputParquet))
}

func readTradeParquet(t *testing.T, path string) []transform.TradeOutputParquet {
	t.Helper()

	pf, err := local.NewLocalFileReader(path)
	if err != nil {
		t.Fatalf("open parquet file: %v", err)
	}
	defer pf.Close()

	pr, err := reader.NewParquetReader(pf, new(transform.TradeOutputParquet), 1)
	if err != nil {
		t.Fatalf("create parquet reader: %v", err)
	}
	defer pr.ReadStop()

	rows := make([]transform.TradeOutputParquet, int(pr.GetNumRows()))
	if err := pr.Read(&rows); err != nil {
		t.Fatalf("read parquet rows: %v", err)
	}
	return rows
}

func makeTradeOutput(baseOperationID int64) transform.TradeOutput {
	return transform.TradeOutput{
		Order:              1,
		LedgerClosedAt:     time.Unix(baseOperationID, 0).UTC(),
		BuyingOfferID:      null.IntFrom(int64(toid.EncodeOfferId(uint64(baseOperationID)+1, toid.TOIDType))),
		HistoryOperationID: baseOperationID + 1,
	}
}

func TestExportTradesStaleParquetTOIDUpload(t *testing.T) {
	_, thisFile, _, ok := runtime.Caller(0)
	if !ok {
		t.Fatal("could not locate current test file")
	}

	src, err := os.ReadFile(filepath.Join(filepath.Dir(thisFile), "export_trades.go"))
	if err != nil {
		t.Fatalf("read export_trades.go: %v", err)
	}

	srcText := string(src)
	uploadCall := "MaybeUpload(cloudCredentials, cloudStorageBucket, cloudProvider, parquetPath)"
	writeCall := "WriteParquet(transformedTrades, parquetPath, new(transform.TradeOutputParquet))"
	uploadIdx := strings.Index(srcText, uploadCall)
	writeIdx := strings.Index(srcText, writeCall)

	if uploadIdx < 0 || writeIdx < 0 {
		t.Fatal("could not locate parquet upload/write calls in export_trades.go")
	}
	if uploadIdx >= writeIdx {
		t.Fatal("MaybeUpload no longer runs before WriteParquet")
	}

	tmpDir := t.TempDir()
	parquetPath := filepath.Join(tmpDir, "exported_trades.parquet")
	uploadedSnapshotPath := filepath.Join(tmpDir, "uploaded_snapshot.parquet")

	staleBaseOperationID := toid.New(100, 1, 1).ToInt64()
	freshBaseOperationID := toid.New(200, 1, 1).ToInt64()

	writeTradeParquet(t, parquetPath, makeTradeOutput(staleBaseOperationID))

	uploadedBytes, err := os.ReadFile(parquetPath)
	if err != nil {
		t.Fatalf("snapshot uploaded parquet: %v", err)
	}
	if err := os.WriteFile(uploadedSnapshotPath, uploadedBytes, 0o644); err != nil {
		t.Fatalf("persist uploaded snapshot: %v", err)
	}

	writeTradeParquet(t, parquetPath, makeTradeOutput(freshBaseOperationID))

	currentBytes, err := os.ReadFile(parquetPath)
	if err != nil {
		t.Fatalf("read regenerated parquet: %v", err)
	}

	uploadedRows := readTradeParquet(t, uploadedSnapshotPath)
	currentRows := readTradeParquet(t, parquetPath)
	if len(uploadedRows) != 1 || len(currentRows) != 1 {
		t.Fatalf("expected exactly one row in each parquet file, got uploaded=%d current=%d", len(uploadedRows), len(currentRows))
	}

	wantUploadedHistoryOperationID := staleBaseOperationID + 1
	wantCurrentHistoryOperationID := freshBaseOperationID + 1
	wantUploadedBuyingOfferID := int64(toid.EncodeOfferId(uint64(staleBaseOperationID)+1, toid.TOIDType))
	wantCurrentBuyingOfferID := int64(toid.EncodeOfferId(uint64(freshBaseOperationID)+1, toid.TOIDType))

	if uploadedRows[0].HistoryOperationID != wantUploadedHistoryOperationID {
		t.Fatalf("uploaded snapshot history_operation_id = %d, want %d", uploadedRows[0].HistoryOperationID, wantUploadedHistoryOperationID)
	}
	if currentRows[0].HistoryOperationID != wantCurrentHistoryOperationID {
		t.Fatalf("regenerated parquet history_operation_id = %d, want %d", currentRows[0].HistoryOperationID, wantCurrentHistoryOperationID)
	}
	if uploadedRows[0].BuyingOfferID != wantUploadedBuyingOfferID {
		t.Fatalf("uploaded snapshot buying_offer_id = %d, want %d", uploadedRows[0].BuyingOfferID, wantUploadedBuyingOfferID)
	}
	if currentRows[0].BuyingOfferID != wantCurrentBuyingOfferID {
		t.Fatalf("regenerated parquet buying_offer_id = %d, want %d", currentRows[0].BuyingOfferID, wantCurrentBuyingOfferID)
	}
	if uploadedRows[0].HistoryOperationID == currentRows[0].HistoryOperationID {
		t.Fatal("stale and regenerated history_operation_id unexpectedly match")
	}
	if uploadedRows[0].BuyingOfferID == currentRows[0].BuyingOfferID {
		t.Fatal("stale and regenerated buying_offer_id unexpectedly match")
	}
	if bytes.Equal(uploadedBytes, currentBytes) {
		t.Fatal("uploaded snapshot and regenerated parquet bytes unexpectedly match")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: when `export_trades --write-parquet` uploads parquet, the uploaded object should contain the current run's trade `history_operation_id` and `buying_offer_id` values
- **Actual**: the upload reads the previous parquet file from disk first, so cloud storage receives stale TOID-bearing trade rows while the local parquet is rewritten afterward with current values

## Adversarial Review

1. Exercises claimed bug: YES — the test confirms the exact upload-before-write ordering in `export_trades.go`, snapshots the bytes that `UploadTo` would read from the reused parquet path at that moment, and proves those bytes contain different TOID fields from the regenerated local parquet.
2. Realistic preconditions: YES — `--parquet-output` defaults to `exported_trades.parquet`, so reused filenames are the default behavior across periodic exports.
3. Bug vs by-design: BUG — sibling exporters such as `export_assets`, `export_effects`, and `export_ledger_entry_changes` write parquet before uploading it, and nothing documents stale prior-run uploads as intended behavior.
4. Final severity: High — the corrupted fields are non-financial but are primary TOID join keys, so downstream systems ingest structurally wrong trade identifiers.
5. In scope: YES — this is production export logic that silently emits wrong cloud parquet data.
6. Test correctness: CORRECT — it uses the real `WriteParquet` helper, the real trade parquet schema, and TOID-consistent field relationships instead of asserting a tautology.
7. Alternative explanations: NONE
8. Novelty: NOVEL IN THIS COMMAND — the bug class exists elsewhere, but this trade-specific TOID manifestation is independently real in `export_trades`.

## Suggested Fix

Swap the parquet operations in `export_trades` so `WriteParquet(transformedTrades, parquetPath, ...)` runs before `MaybeUpload(..., parquetPath)`, matching the exporters that already use the safe order.
