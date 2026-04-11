# H005: `export_trades` can upload stale `history_operation_id` and synthetic offer IDs before parquet generation

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: Medium
**Impact**: silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_trades --write-parquet` uploads parquet, the remote file should contain the current run's `history_operation_id` values and any newly generated synthetic `buying_offer_id` TOIDs. Cloud readers should not observe a previous ledger range's trade IDs after a successful rerun.

## Mechanism

`export_trades` calls `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. Reusing the same local parquet filename therefore uploads a stale trade parquet object first, and only afterward regenerates the local file with the current run's `history_operation_id` and synthetic offer IDs derived from the trade operation TOIDs.

## Trigger

1. Run `export_trades --write-parquet` with cloud upload enabled and a fixed parquet output path.
2. Re-run it for a different ledger range using the same parquet path.
3. Compare the remote parquet's `history_operation_id` / `buying_offer_id` values with the current local parquet output.

## Target Code

- `cmd/export_trades.go:66-70` — uploads `parquetPath` before calling `WriteParquet(...)`.
- `cmd/command_utils.go:148-180` — the parquet file is only regenerated inside `WriteParquet(...)`.
- `internal/transform/schema_parquet.go:218-244` — trade parquet carries TOID-bearing `history_operation_id` plus offer-ID columns.
- `internal/transform/trade.go:116-120,152` — trade rows derive `buying_offer_id` and `history_operation_id` from operation TOIDs.

## Evidence

The command correctly accumulates freshly transformed trade rows in memory, but those new IDs do not reach disk until the final parquet write. Uploading first therefore sends an earlier parquet snapshot whenever the output path is reused, which is especially dangerous because the stale file still looks structurally valid.

## Anti-Evidence

If the parquet path is unique per run, the bad ordering turns into a loud missing-file problem instead of silent stale data. The hypothesis depends on a reused parquet output path, but that is a common pattern for periodic exports that overwrite the same local staging filename.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (success/002 covers `export_operations`, not `export_trades`)

### Trace Summary

Traced `export_trades.go:68-71` confirming `MaybeUpload(parquetPath)` is called before `WriteParquet(transformedTrades, parquetPath, ...)`. Then traced `MaybeUpload` at `command_utils.go:123-146` confirming it immediately dispatches to `GCS.UploadTo()` which opens the on-disk file at call time. `WriteParquet` at `command_utils.go:162-180` is the first function that creates the current run's parquet contents. This is the identical bug pattern confirmed in `export_operations` (success/002) but in a separate command.

### Code Paths Examined

- `cmd/export_trades.go:68-71` — parquet block calls `MaybeUpload` on line 69 then `WriteParquet` on line 70 (upload-before-write confirmed)
- `cmd/command_utils.go:123-146` — `MaybeUpload` immediately invokes `cloudStorage.UploadTo()` which opens the path on disk
- `cmd/command_utils.go:162-180` — `WriteParquet` creates the parquet file writer and writes current run data
- `internal/transform/trade.go:33,119,152` — trade rows derive `HistoryOperationID` and `BuyingOfferID` from TOID-encoded operation IDs
- `cmd/export_assets.go:80-81` — sibling command uses correct write-then-upload order (confirming this is a bug, not design)
- `cmd/export_effects.go:67-68` — another sibling with correct write-then-upload order
- `cmd/export_ledger_entry_changes.go:371-372` — another sibling with correct write-then-upload order

### Findings

The bug is real. Four export commands have the upload-before-write ordering bug: `export_trades`, `export_operations`, `export_ledgers`, and `export_transactions`. Three commands have the correct write-then-upload order: `export_assets`, `export_effects`, and `export_ledger_entry_changes`. The default parquet path from `AddArchiveFlags` is `exported_trades.parquet`, which is reused across runs, making the stale-upload scenario the default behavior.

The trade-specific impact is that TOID-derived fields (`history_operation_id` at trade.go:152, `buying_offer_id` at trade.go:119) in the uploaded parquet will belong to the previous ledger range, not the current one. These are the primary join keys for trade data in downstream BigQuery analytics.

### PoC Guidance

- **Test file**: `cmd/export_trades_poc_test.go` (new file, following pattern from success/002's PoC)
- **Setup**: Read `export_trades.go` source to verify call ordering; create a temp parquet path with stale trade data
- **Steps**:
  1. Assert `MaybeUpload` appears before `WriteParquet` in the source of `export_trades.go` (source ordering test)
  2. Write a stale trade parquet file with `WriteParquet` using ledger-100 TOIDs, then read it back to confirm stale `HistoryOperationID` values
  3. Write a fresh trade parquet file to the same path using ledger-200 TOIDs, read it back, and assert the values differ from the stale file
- **Assertion**: The stale parquet `history_operation_id` differs from the freshly written one, proving upload-before-write sends wrong data

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: cmd/export_trades_poc_test.go
**Test Name**: "TestExportTradesStaleParquetTOIDUpload"
**Test Language**: Go

### Demonstration

The test confirms that `export_trades.go` calls `MaybeUpload(parquetPath)` before `WriteParquet(transformedTrades, parquetPath, ...)` within the same `if commonArgs.WriteParquet` block. It then writes a stale trade parquet with ledger-100 TOID-derived `history_operation_id` (429496733697) and `buying_offer_id` (4611686447924121602), overwrites the same path with ledger-200 TOIDs (858993463297 and 4611686877420851202), and proves the two files contain entirely different TOID values — meaning the upload-before-write ordering sends the wrong ledger range's trade IDs to cloud storage.

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

	"github.com/guregu/null"
	"github.com/stellar/stellar-etl/v2/internal/toid"
	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/reader"
	"github.com/xitongsys/parquet-go/writer"
)

// writeTradeParquet writes trade records to a parquet file at the given path.
func writeTradeParquet(t *testing.T, trades []transform.SchemaParquet, path string) {
	t.Helper()
	pf, err := local.NewLocalFileWriter(path)
	if err != nil {
		t.Fatalf("create parquet file: %v", err)
	}
	defer pf.Close()

	pw, err := writer.NewParquetWriter(pf, new(transform.TradeOutputParquet), 1)
	if err != nil {
		t.Fatalf("create parquet writer: %v", err)
	}
	defer pw.WriteStop()

	for _, rec := range trades {
		if err := pw.Write(rec.ToParquet()); err != nil {
			t.Fatalf("write parquet record: %v", err)
		}
	}
}

// readTradeParquet reads all TradeOutputParquet records from a parquet file.
func readTradeParquet(t *testing.T, path string) []transform.TradeOutputParquet {
	t.Helper()
	pf, err := local.NewLocalFileReader(path)
	if err != nil {
		t.Fatalf("open parquet file: %v", err)
	}
	defer pf.Close()

	pr, err := reader.NewParquetReader(pf, new(transform.TradeOutputParquet), 4)
	if err != nil {
		t.Fatalf("create parquet reader: %v", err)
	}
	defer pr.ReadStop()

	numRows := int(pr.GetNumRows())
	rows := make([]transform.TradeOutputParquet, numRows)
	if err := pr.Read(&rows); err != nil {
		t.Fatalf("read parquet rows: %v", err)
	}
	return rows
}

// TestExportTradesStaleParquetTOIDUpload demonstrates that export_trades uploads
// the parquet file before regenerating it. Because MaybeUpload is called before
// WriteParquet, a reused parquet path sends the previous run's TOID-derived
// history_operation_id and buying_offer_id values to cloud storage.
func TestExportTradesStaleParquetTOIDUpload(t *testing.T) {
	// ---------------------------------------------------------------
	// Part 1: Verify source ordering — MaybeUpload before WriteParquet
	// ---------------------------------------------------------------
	_, thisFile, _, _ := runtime.Caller(0)
	cmdDir := filepath.Dir(thisFile)
	src, err := os.ReadFile(filepath.Join(cmdDir, "export_trades.go"))
	if err != nil {
		t.Fatalf("read export_trades.go: %v", err)
	}

	// Only consider the WriteParquet block (inside "if commonArgs.WriteParquet")
	srcStr := string(src)
	uploadIdx := strings.Index(srcStr, "MaybeUpload(cloudCredentials, cloudStorageBucket, cloudProvider, parquetPath)")
	writeIdx := strings.Index(srcStr, "WriteParquet(transformedTrades, parquetPath,")

	if uploadIdx < 0 || writeIdx < 0 {
		t.Fatal("could not locate MaybeUpload or WriteParquet calls in export_trades.go")
	}

	if uploadIdx >= writeIdx {
		t.Fatal("MaybeUpload is NOT before WriteParquet — bug may have been fixed")
	}

	// Confirm both calls are inside the same WriteParquet block
	blockStart := strings.LastIndex(srcStr[:uploadIdx], "if commonArgs.WriteParquet {")
	if blockStart < 0 {
		t.Fatal("MaybeUpload(parquetPath) is not inside a WriteParquet block")
	}
	blockAfterUpload := srcStr[blockStart:]
	if !strings.Contains(blockAfterUpload, "WriteParquet(transformedTrades") {
		t.Fatal("WriteParquet(transformedTrades) is not in the same block as MaybeUpload(parquetPath)")
	}

	t.Log("CONFIRMED: MaybeUpload(parquetPath) appears before WriteParquet in export_trades.go")

	// ---------------------------------------------------------------
	// Part 2: Stale parquet retains old TOID-based trade IDs
	// ---------------------------------------------------------------
	tmpDir := t.TempDir()
	parquetPath := filepath.Join(tmpDir, "exported_trades.parquet")

	// Simulate a previous run at ledger 100, tx 1, op 1
	staleLedger := int32(100)
	staleOpID := toid.New(staleLedger, 1, 1).ToInt64()
	staleBuyingOfferID := toid.EncodeOfferId(uint64(staleOpID)+1, toid.TOIDType)

	staleTrades := []transform.SchemaParquet{
		transform.TradeOutput{
			HistoryOperationID: staleOpID,
			BuyingOfferID:      newNullInt(staleBuyingOfferID),
		},
	}
	writeTradeParquet(t, staleTrades, parquetPath)

	// Read the stale file back and snapshot its bytes
	staleRows := readTradeParquet(t, parquetPath)
	if len(staleRows) != 1 {
		t.Fatalf("expected 1 stale row, got %d", len(staleRows))
	}
	staleBytes, err := os.ReadFile(parquetPath)
	if err != nil {
		t.Fatalf("read stale parquet: %v", err)
	}

	t.Logf("Stale parquet: history_operation_id=%d (ledger=%d), buying_offer_id=%d",
		staleRows[0].HistoryOperationID, staleLedger, staleRows[0].BuyingOfferID)

	// ---------------------------------------------------------------
	// Part 3: Fresh run at ledger 200 — same path, different TOIDs
	// ---------------------------------------------------------------
	freshLedger := int32(200)
	freshOpID := toid.New(freshLedger, 1, 1).ToInt64()
	freshBuyingOfferID := toid.EncodeOfferId(uint64(freshOpID)+1, toid.TOIDType)

	freshTrades := []transform.SchemaParquet{
		transform.TradeOutput{
			HistoryOperationID: freshOpID,
			BuyingOfferID:      newNullInt(freshBuyingOfferID),
		},
	}
	writeTradeParquet(t, freshTrades, parquetPath)

	freshRows := readTradeParquet(t, parquetPath)
	if len(freshRows) != 1 {
		t.Fatalf("expected 1 fresh row, got %d", len(freshRows))
	}
	freshBytes, err := os.ReadFile(parquetPath)
	if err != nil {
		t.Fatalf("read fresh parquet: %v", err)
	}

	t.Logf("Fresh parquet: history_operation_id=%d (ledger=%d), buying_offer_id=%d",
		freshRows[0].HistoryOperationID, freshLedger, freshRows[0].BuyingOfferID)

	// ---------------------------------------------------------------
	// Part 4: Assert stale and fresh data differ — proving the bug
	// ---------------------------------------------------------------

	// The TOID-derived IDs must differ between the two ledger ranges
	if staleRows[0].HistoryOperationID == freshRows[0].HistoryOperationID {
		t.Fatal("stale and fresh history_operation_id are identical — test is invalid")
	}
	if staleRows[0].BuyingOfferID == freshRows[0].BuyingOfferID {
		t.Fatal("stale and fresh buying_offer_id are identical — test is invalid")
	}

	// The raw file bytes must differ
	if bytes.Equal(staleBytes, freshBytes) {
		t.Fatal("stale and fresh parquet bytes are identical — test is invalid")
	}

	// Because export_trades calls MaybeUpload BEFORE WriteParquet,
	// any upload triggered at line 69 would read staleBytes from disk,
	// not freshBytes. The fresh data doesn't exist on disk until line 70.
	t.Logf("BUG DEMONSTRATED: export_trades uploads stale parquet (history_operation_id=%d, buying_offer_id=%d) "+
		"before regenerating the file with current data (history_operation_id=%d, buying_offer_id=%d). "+
		"Stale file is %d bytes, fresh file is %d bytes — they differ.",
		staleRows[0].HistoryOperationID, staleRows[0].BuyingOfferID,
		freshRows[0].HistoryOperationID, freshRows[0].BuyingOfferID,
		len(staleBytes), len(freshBytes))
}

func newNullInt(v int64) null.Int {
	return null.IntFrom(v)
}
```

### Test Output

```
=== RUN   TestExportTradesStaleParquetTOIDUpload
    export_trades_poc_test.go:104: CONFIRMED: MaybeUpload(parquetPath) appears before WriteParquet in export_trades.go
    export_trades_poc_test.go:135: Stale parquet: history_operation_id=429496733697 (ledger=100), buying_offer_id=4611686447924121602
    export_trades_poc_test.go:162: Fresh parquet: history_operation_id=858993463297 (ledger=200), buying_offer_id=4611686877420851202
    export_trades_poc_test.go:185: BUG DEMONSTRATED: export_trades uploads stale parquet (history_operation_id=429496733697, buying_offer_id=4611686447924121602) before regenerating the file with current data (history_operation_id=858993463297, buying_offer_id=4611686877420851202). Stale file is 5213 bytes, fresh file is 5213 bytes — they differ.
--- PASS: TestExportTradesStaleParquetTOIDUpload (0.01s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.565s
```
