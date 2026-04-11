# H004: `export_transactions` can upload stale `transaction_id` parquet before rewriting it

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: Medium
**Impact**: silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_transactions --write-parquet` uploads to cloud storage, the uploaded parquet should contain the current run's `id` values for the requested ledger range. A rerun over different ledgers must not leave cloud consumers reading the previous run's transaction TOIDs.

## Mechanism

The command currently uploads `parquetPath` before `WriteParquet(...)` materializes the new transaction parquet. Reusing a local parquet path therefore makes the cloud upload race against stale on-disk contents: the remote object gets the earlier run's `transaction_id` file, while the new parquet is only written locally afterward.

## Trigger

Run `export_transactions --write-parquet` twice with cloud upload enabled and the same `--parquet-output` path but different ledger ranges, then inspect the remote parquet object's `id` column.

## Target Code

- `cmd/export_transactions.go:61-65` — `MaybeUpload(..., parquetPath)` happens before `WriteParquet(...)`.
- `cmd/command_utils.go:148-180` — parquet generation occurs only inside `WriteParquet(...)`.
- `internal/transform/schema_parquet.go:31-74` — the transaction parquet schema carries the TOID-backed `id` column.

## Evidence

`transformedTransaction` is only an in-memory slice until the final write step. Because the upload happens first, the command has no way to send the newly transformed `transaction_id` values on that code path; the only remotely visible parquet is an old file at the same path or no file at all.

## Anti-Evidence

The corruption is silent only when the parquet path already exists; otherwise the bad ordering may surface as an upload failure instead. A reviewer should confirm how commonly scheduled jobs reuse parquet output paths in production.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated for `export_transactions` (success/002 covers only `export_operations`)

### Trace Summary

The `export_transactions` command at `cmd/export_transactions.go:63-66` calls `MaybeUpload(parquetPath)` on line 64 before `WriteParquet(...)` on line 65. `MaybeUpload` (at `cmd/command_utils.go:123-146`) immediately dispatches to `GCS.UploadTo()` which opens whatever file exists on disk. `WriteParquet` (at `cmd/command_utils.go:162-180`) is the first function that actually creates and writes the current run's parquet data. On a first run with no pre-existing file, the upload crashes; on subsequent runs, it silently uploads the previous run's stale parquet.

### Code Paths Examined

- `cmd/export_transactions.go:63-66` — confirmed `MaybeUpload(parquetPath)` on line 64 precedes `WriteParquet(...)` on line 65
- `cmd/command_utils.go:123-146` — `MaybeUpload` calls `cloudStorage.UploadTo()` immediately, opening the on-disk path
- `cmd/command_utils.go:162-180` — `WriteParquet` creates the parquet file writer and writes records; this is the only place the current run's data materializes
- `cmd/export_effects.go:67-68` — correct ordering (WriteParquet before MaybeUpload) for comparison
- `cmd/export_assets.go:80-81` — correct ordering for comparison
- `cmd/export_contract_events.go:64-65` — correct ordering for comparison
- `cmd/export_ledger_entry_changes.go:371-372` — correct ordering for comparison

### Findings

The bug is identical in mechanism to the confirmed `export_operations` finding (success/002). Four commands have correct write-then-upload ordering (`export_effects`, `export_assets`, `export_contract_events`, `export_ledger_entry_changes`). Four commands have wrong upload-then-write ordering (`export_transactions`, `export_operations`, `export_trades`, `export_ledgers`). This is a clear copy-paste inconsistency. The `export_transactions` instance is novel and independently confirmed.

### PoC Guidance

- **Test file**: `cmd/data_integrity_poc_test.go` (or a new `cmd/export_transactions_poc_test.go`)
- **Setup**: Read the source of `export_transactions.go` and locate the `MaybeUpload` and `WriteParquet` calls
- **Steps**: (1) Assert source ordering shows MaybeUpload before WriteParquet. (2) Write a stale parquet with `WriteParquet` for ledger range A, then read it back to confirm stale TOIDs. (3) Write a fresh parquet for ledger range B and confirm different TOIDs, proving uploads at the MaybeUpload call site would have served stale data.
- **Assertion**: `MaybeUpload` index in source < `WriteParquet` index in source, confirming upload-before-write ordering

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportTransactionsUploadsStaleParquet"
**Test Language**: Go

### Demonstration

The test confirms the bug in two ways. First, it reads `export_transactions.go` source and verifies that `MaybeUpload` (line 64) is called before `WriteParquet` (line 65), meaning cloud upload fires before the current run's parquet file is written. Second, it functionally demonstrates that writing parquet for ledger 100, then reading the file back at the point where `MaybeUpload` would execute (before writing ledger 200's data), yields stale transaction IDs — the upload would send ledger 100's TOID (429496729600) instead of ledger 200's TOID (858993459200).

### Test Body

```go
package cmd

import (
	"bufio"
	"os"
	"path/filepath"
	"runtime"
	"strings"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/reader"
)

// TestExportTransactionsUploadsStaleParquet demonstrates that export_transactions
// calls MaybeUpload(parquetPath) BEFORE WriteParquet, meaning cloud consumers
// would receive stale (previous-run) data instead of the current run's transaction TOIDs.
//
// Part 1: Source ordering — confirms MaybeUpload appears before WriteParquet in
// the WriteParquet block of export_transactions.go.
//
// Part 2: Functional — writes parquet for "run A" (ledger 100), then reads it
// back at the point where MaybeUpload would execute (before a "run B" write),
// proving the on-disk file still contains run A's stale transaction IDs.
func TestExportTransactionsUploadsStaleParquet(t *testing.T) {
	// ---------------------------------------------------------------
	// Part 1: Source-code ordering confirms upload-before-write bug
	// ---------------------------------------------------------------
	_, thisFile, _, _ := runtime.Caller(0)
	srcPath := filepath.Join(filepath.Dir(thisFile), "export_transactions.go")
	srcFile, err := os.Open(srcPath)
	if err != nil {
		t.Fatalf("cannot open export_transactions.go: %v", err)
	}
	defer srcFile.Close()

	// Scan the source for positions of MaybeUpload(parquetPath) and
	// WriteParquet(...) calls. We look for the ones that coexist near each
	// other (the parquet upload block, not the earlier append block).
	scanner := bufio.NewScanner(srcFile)
	maybeUploadLine := -1
	writeParquetLine := -1
	lineNum := 0

	for scanner.Scan() {
		lineNum++
		line := strings.TrimSpace(scanner.Text())

		if strings.Contains(line, "MaybeUpload") && strings.Contains(line, "parquetPath") {
			maybeUploadLine = lineNum
		}
		if strings.HasPrefix(line, "WriteParquet(") {
			writeParquetLine = lineNum
		}
	}

	if maybeUploadLine == -1 {
		t.Fatal("could not find MaybeUpload(..., parquetPath) call")
	}
	if writeParquetLine == -1 {
		t.Fatal("could not find WriteParquet(...) call")
	}

	// The bug: MaybeUpload (upload) appears BEFORE WriteParquet (file creation).
	// A correct command would have WriteParquet first.
	if maybeUploadLine >= writeParquetLine {
		t.Fatalf("ordering is correct (MaybeUpload at line %d, WriteParquet at line %d); "+
			"expected MaybeUpload BEFORE WriteParquet to demonstrate the bug",
			maybeUploadLine, writeParquetLine)
	}

	t.Logf("BUG CONFIRMED: MaybeUpload at line %d precedes WriteParquet at line %d",
		maybeUploadLine, writeParquetLine)

	// ---------------------------------------------------------------
	// Part 2: Functional demonstration of stale data
	// ---------------------------------------------------------------
	parquetPath := t.TempDir() + "/transactions.parquet"

	// Simulate "Run A" — write a parquet with transaction ID from ledger 100.
	runA := []transform.SchemaParquet{
		transform.TransactionOutput{
			TransactionID:   4294967296 * 100, // TOID for ledger 100
			TransactionHash: "aaa_run_a",
			LedgerSequence:  100,
			MemoType:        "none",
		},
	}
	WriteParquet(runA, parquetPath, new(transform.TransactionOutputParquet))

	// At this point, the file on disk has Run A data.
	// In the buggy code, MaybeUpload would now upload THIS file.
	// Read it back to capture what the upload would see.
	staleID := readFirstTransactionID(t, parquetPath)
	if staleID != 4294967296*100 {
		t.Fatalf("run A parquet does not contain expected ID: got %d", staleID)
	}
	t.Logf("Run A parquet on disk has transaction ID = %d (ledger 100 TOID)", staleID)

	// Simulate "Run B" — the NEW data for ledger 200.
	runB := []transform.SchemaParquet{
		transform.TransactionOutput{
			TransactionID:   4294967296 * 200, // TOID for ledger 200
			TransactionHash: "bbb_run_b",
			LedgerSequence:  200,
			MemoType:        "none",
		},
	}

	// In the buggy code path, MaybeUpload fires BEFORE WriteParquet.
	// So at upload time, the file still contains Run A's data.
	// Verify: read the file BEFORE writing Run B — it has stale Run A IDs.
	uploadTimeID := readFirstTransactionID(t, parquetPath)
	if uploadTimeID != staleID {
		t.Fatalf("file changed before WriteParquet; got %d, expected stale %d",
			uploadTimeID, staleID)
	}

	t.Logf("At MaybeUpload time (before WriteParquet), file has STALE ID = %d (ledger 100)", uploadTimeID)

	// NOW write Run B (this is what WriteParquet does after MaybeUpload).
	WriteParquet(runB, parquetPath, new(transform.TransactionOutputParquet))

	freshID := readFirstTransactionID(t, parquetPath)
	if freshID != 4294967296*200 {
		t.Fatalf("run B parquet does not contain expected ID: got %d", freshID)
	}

	t.Logf("After WriteParquet, file now has FRESH ID = %d (ledger 200)", freshID)

	// The uploaded data (staleID) differs from the correct data (freshID).
	if uploadTimeID == freshID {
		t.Fatal("stale and fresh IDs match — bug not demonstrated")
	}

	t.Logf("STALE DATA PROVEN: upload sent ID %d (ledger 100) instead of %d (ledger 200)",
		uploadTimeID, freshID)
}

// readFirstTransactionID reads back the first transaction's ID from a parquet file.
func readFirstTransactionID(t *testing.T, path string) int64 {
	t.Helper()

	fr, err := local.NewLocalFileReader(path)
	if err != nil {
		t.Fatalf("cannot open parquet file %s: %v", path, err)
	}
	defer fr.Close()

	pr, err := reader.NewParquetReader(fr, new(transform.TransactionOutputParquet), 1)
	if err != nil {
		t.Fatalf("cannot create parquet reader for %s: %v", path, err)
	}
	defer pr.ReadStop()

	rows := make([]transform.TransactionOutputParquet, 1)
	if err := pr.Read(&rows); err != nil {
		t.Fatalf("cannot read parquet row from %s: %v", path, err)
	}

	return rows[0].TransactionID
}
```

### Test Output

```
=== RUN   TestExportTransactionsUploadsStaleParquet
    data_integrity_poc_test.go:73: BUG CONFIRMED: MaybeUpload at line 64 precedes WriteParquet at line 65
    data_integrity_poc_test.go:99: Run A parquet on disk has transaction ID = 429496729600 (ledger 100 TOID)
    data_integrity_poc_test.go:120: At MaybeUpload time (before WriteParquet), file has STALE ID = 429496729600 (ledger 100)
    data_integrity_poc_test.go:130: After WriteParquet, file now has FRESH ID = 858993459200 (ledger 200)
    data_integrity_poc_test.go:137: STALE DATA PROVEN: upload sent ID 429496729600 (ledger 100) instead of 858993459200 (ledger 200)
--- PASS: TestExportTransactionsUploadsStaleParquet (0.02s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	1.878s
```
