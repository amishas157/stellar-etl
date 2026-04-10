# H001: Parquet upload happens before current batch is written

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `--write-parquet` and GCS upload are both enabled, each export command should serialize the current ledger range to the local parquet file first, then upload that exact file. The JSON and Parquet objects for a given run should represent the same ledgers and the same row set.

## Mechanism

`export_ledgers`, `export_transactions`, `export_operations`, and `export_trades` call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. On a first run, the upload path does not exist yet, so upload fails before any parquet file is created. On a repeated run that reuses the same local path, `UploadTo` opens the stale parquet from the previous export, uploads it, and deletes it before the new batch is written; the fresh parquet is then created locally but never uploaded.

## Trigger

1. Run `export_ledgers`, `export_transactions`, `export_operations`, or `export_trades` with `--write-parquet` and `--cloud-provider gcp`.
2. Use a parquet output path that either does not exist yet (first run) or still contains the previous run's parquet file.
3. Observe that GCS either receives no parquet object for the current batch or receives the previous batch's parquet while JSON reflects the current batch.

## Target Code

- `cmd/export_ledgers.go:Run:68-73` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/export_transactions.go:Run:61-66` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/export_operations.go:Run:61-66` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/export_trades.go:Run:66-70` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/upload_to_gcs.go:UploadTo:32-35,47-73` — opens the existing local file and deletes it after upload

## Evidence

The four commands above all use the reversed order `MaybeUpload(parquetPath)` then `WriteParquet(...)`. `UploadTo` reads whatever file is already present at `path`, verifies the object exists in GCS, and then removes the local file via `deleteLocalFiles(path)`.

## Anti-Evidence

Other exporters in the same package use the correct order. `export_assets`, `export_effects`, `export_contract_events`, and `export_ledger_entry_changes` all write the parquet file before uploading it, which suggests the intended behavior is well understood elsewhere in the subsystem.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the parquet write-then-upload sequence across all 10 export commands. Four commands (`export_ledgers`, `export_transactions`, `export_operations`, `export_trades`) call `MaybeUpload(parquetPath)` before `WriteParquet()`, while six other commands use the correct order. `UploadTo` in `upload_to_gcs.go` calls `os.Open(path)` to read the local file, uploads it to GCS, then calls `deleteLocalFiles(path)` to remove it. When the parquet file does not yet exist (because `WriteParquet` hasn't run), `os.Open` fails and `MaybeUpload` fatals. When a stale file exists from a previous run, the stale data is uploaded and deleted, and the fresh file written by `WriteParquet` is never uploaded.

### Code Paths Examined

- `cmd/export_ledgers.go:70-73` — `MaybeUpload(parquetPath)` on line 71, `WriteParquet(...)` on line 72; reversed order confirmed
- `cmd/export_transactions.go:63-66` — same reversed order: `MaybeUpload` line 64, `WriteParquet` line 65
- `cmd/export_operations.go:63-66` — same reversed order: `MaybeUpload` line 64, `WriteParquet` line 65
- `cmd/export_trades.go:68-71` — same reversed order: `MaybeUpload` line 69, `WriteParquet` line 70
- `cmd/export_effects.go:66-69` — CORRECT order: `WriteParquet` line 67, `MaybeUpload` line 68
- `cmd/export_assets.go:79-82` — CORRECT order: `WriteParquet` line 80, `MaybeUpload` line 81
- `cmd/command_utils.go:123-146` — `MaybeUpload` delegates to `UploadTo`; fatals on error
- `cmd/upload_to_gcs.go:25-74` — `UploadTo` opens local file (line 32), uploads to GCS, then calls `deleteLocalFiles(path)` (line 71)

### Findings

Two distinct failure modes confirmed:

1. **First run (no pre-existing parquet file)**: `MaybeUpload` → `UploadTo` → `os.Open(parquetPath)` fails with "file not found" → error returned → `MaybeUpload` calls `cmdLogger.Fatalf(...)` → **process terminates**. The parquet file is never written and never uploaded. The JSON file was already uploaded successfully on the prior `MaybeUpload(path)` call, so JSON data reaches GCS but parquet data does not.

2. **Subsequent run (stale parquet file exists at same path)**: `MaybeUpload` → `UploadTo` opens the stale file → uploads **previous batch's data** to GCS → `deleteLocalFiles` removes the stale file → control returns → `WriteParquet` writes the **current batch** to the same local path. The current batch's parquet is created locally but never uploaded. GCS now holds stale parquet data that does not match the current JSON export.

This matches Investigation Pattern 3 (export command consistency): the 4 affected commands deviate from the 6 correct commands in the ordering of WriteParquet vs MaybeUpload calls.

Severity confirmed as **High** — this produces structural data corruption where the parquet object in GCS contains data from a different ledger range than the JSON object from the same export run.

### PoC Guidance

- **Test file**: `cmd/export_ledgers_test.go` (or a new integration-level test file)
- **Setup**: Create a mock GCS upload mechanism or use filesystem observation. Run `export_ledgers` twice with `--write-parquet` and a mock cloud provider, using the same `parquetPath` both times but different ledger ranges.
- **Steps**:
  1. First invocation: call the ledgers command Run function with `--write-parquet` and `--cloud-provider gcp`. Observe that `MaybeUpload` is called with `parquetPath` before the file exists.
  2. Second invocation: place a known parquet file at `parquetPath`, then run the command again with a different ledger range. Observe which file content is uploaded.
- **Assertion**: Verify that the file content uploaded to GCS on the second run matches the *previous* run's data (stale), not the current run's transformed data. Alternatively, verify that swapping the two lines (`WriteParquet` before `MaybeUpload`) causes the correct data to be uploaded.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestParquetUploadPrecedesWrite" and "TestUploadToFailsWhenParquetFileNotYetWritten"
**Test Language**: Go

### Demonstration

Two complementary tests prove the hypothesis. `TestParquetUploadPrecedesWrite` performs source analysis on all 6 export commands with parquet support, confirming that the 4 affected commands (`export_ledgers`, `export_transactions`, `export_operations`, `export_trades`) call `MaybeUpload(parquetPath)` before `WriteParquet()`, while the 2 correct commands (`export_effects`, `export_assets`) use the correct order. `TestUploadToFailsWhenParquetFileNotYetWritten` demonstrates the runtime consequence: when `UploadTo` is called with a path that doesn't exist yet (because `WriteParquet` hasn't run), `os.Open()` fails with "failed to open file", which in production triggers `cmdLogger.Fatalf` and terminates the process.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"runtime"
	"strings"
	"testing"
)

func sourceDir() string {
	_, filename, _, _ := runtime.Caller(0)
	return filepath.Dir(filename)
}

// TestParquetUploadPrecedesWrite demonstrates that four export commands
// call MaybeUpload(parquetPath) BEFORE WriteParquet(), meaning the parquet
// file either doesn't exist yet (first run → fatal) or contains stale data
// from a previous run (subsequent runs → wrong data uploaded to GCS).
//
// Correct commands (export_effects, export_assets) call WriteParquet first.
func TestParquetUploadPrecedesWrite(t *testing.T) {
	type testCase struct {
		file        string
		expectBuggy bool // true = MaybeUpload before WriteParquet (the bug)
	}

	cases := []testCase{
		{"export_ledgers.go", true},
		{"export_transactions.go", true},
		{"export_operations.go", true},
		{"export_trades.go", true},
		{"export_effects.go", false},
		{"export_assets.go", false},
	}

	for _, tc := range cases {
		t.Run(tc.file, func(t *testing.T) {
			content, err := os.ReadFile(filepath.Join(sourceDir(), tc.file))
			if err != nil {
				t.Fatalf("Failed to read %s: %v", tc.file, err)
			}

			lines := strings.Split(string(content), "\n")
			maybeUploadLine := -1
			writeParquetLine := -1

			// Scan all `if commonArgs.WriteParquet` blocks; keep the one
			// that contains both MaybeUpload and WriteParquet (the upload block,
			// not the in-loop data-collection block).
			for i := 0; i < len(lines); i++ {
				trimmed := strings.TrimSpace(lines[i])
				if !(strings.Contains(trimmed, "commonArgs.WriteParquet") && strings.HasPrefix(trimmed, "if ")) {
					continue
				}
				// Found a candidate block — scan to its closing brace
				blockUpload := -1
				blockWrite := -1
				for j := i + 1; j < len(lines); j++ {
					inner := strings.TrimSpace(lines[j])
					if strings.Contains(inner, "MaybeUpload(") && blockUpload == -1 {
						blockUpload = j + 1
					}
					if strings.Contains(inner, "WriteParquet(") && blockWrite == -1 {
						blockWrite = j + 1
					}
					if inner == "}" {
						break
					}
				}
				if blockUpload != -1 && blockWrite != -1 {
					maybeUploadLine = blockUpload
					writeParquetLine = blockWrite
					break
				}
			}

			if maybeUploadLine == -1 || writeParquetLine == -1 {
				t.Fatalf("Could not find a WriteParquet block containing both MaybeUpload and WriteParquet in %s", tc.file)
			}

			uploadFirst := maybeUploadLine < writeParquetLine

			if tc.expectBuggy {
				if !uploadFirst {
					t.Errorf("Expected MaybeUpload before WriteParquet in %s, but found MaybeUpload=%d, WriteParquet=%d",
						tc.file, maybeUploadLine, writeParquetLine)
				}
				t.Logf("BUG CONFIRMED in %s: MaybeUpload(parquetPath) on line %d precedes WriteParquet on line %d",
					tc.file, maybeUploadLine, writeParquetLine)
			} else {
				if uploadFirst {
					t.Errorf("Expected correct order in %s, but MaybeUpload=%d precedes WriteParquet=%d",
						tc.file, maybeUploadLine, writeParquetLine)
				}
				t.Logf("CORRECT ORDER in %s: WriteParquet on line %d precedes MaybeUpload on line %d",
					tc.file, writeParquetLine, maybeUploadLine)
			}
		})
	}
}

// TestUploadToFailsWhenParquetFileNotYetWritten shows the runtime consequence:
// UploadTo calls os.Open(path) as its first I/O operation. When MaybeUpload
// is called before WriteParquet, the parquet file doesn't exist, so os.Open
// fails and MaybeUpload would call Fatalf, terminating the export process.
func TestUploadToFailsWhenParquetFileNotYetWritten(t *testing.T) {
	gcs := &GCS{gcsCredentialsPath: "", gcsBucket: "test-bucket"}
	nonExistentPath := filepath.Join(t.TempDir(), "not-yet-written.parquet")

	err := gcs.UploadTo("", "test-bucket", nonExistentPath)
	if err == nil {
		t.Fatal("Expected error when uploading non-existent file, but got nil")
	}

	if !strings.Contains(err.Error(), "failed to open file") {
		t.Fatalf("Expected 'failed to open file' error, got: %v", err)
	}

	t.Logf("UploadTo fails with: %v", err)
	t.Log("This is the first-run consequence: MaybeUpload(parquetPath) is called before WriteParquet(),")
	t.Log("so the parquet file does not exist yet and UploadTo fails at os.Open()")
}
```

### Test Output

```
=== RUN   TestParquetUploadPrecedesWrite
=== RUN   TestParquetUploadPrecedesWrite/export_ledgers.go
    data_integrity_poc_test.go:89: BUG CONFIRMED in export_ledgers.go: MaybeUpload(parquetPath) on line 71 precedes WriteParquet on line 72
=== RUN   TestParquetUploadPrecedesWrite/export_transactions.go
    data_integrity_poc_test.go:89: BUG CONFIRMED in export_transactions.go: MaybeUpload(parquetPath) on line 64 precedes WriteParquet on line 65
=== RUN   TestParquetUploadPrecedesWrite/export_operations.go
    data_integrity_poc_test.go:89: BUG CONFIRMED in export_operations.go: MaybeUpload(parquetPath) on line 64 precedes WriteParquet on line 65
=== RUN   TestParquetUploadPrecedesWrite/export_trades.go
    data_integrity_poc_test.go:89: BUG CONFIRMED in export_trades.go: MaybeUpload(parquetPath) on line 69 precedes WriteParquet on line 70
=== RUN   TestParquetUploadPrecedesWrite/export_effects.go
    data_integrity_poc_test.go:96: CORRECT ORDER in export_effects.go: WriteParquet on line 67 precedes MaybeUpload on line 68
=== RUN   TestParquetUploadPrecedesWrite/export_assets.go
    data_integrity_poc_test.go:96: CORRECT ORDER in export_assets.go: WriteParquet on line 80 precedes MaybeUpload on line 81
--- PASS: TestParquetUploadPrecedesWrite (0.00s)
=== RUN   TestUploadToFailsWhenParquetFileNotYetWritten
    data_integrity_poc_test.go:120: UploadTo fails with: failed to open file .../not-yet-written.parquet: open .../not-yet-written.parquet: no such file or directory
    data_integrity_poc_test.go:121: This is the first-run consequence: MaybeUpload(parquetPath) is called before WriteParquet(),
    data_integrity_poc_test.go:122: so the parquet file does not exist yet and UploadTo fails at os.Open()
--- PASS: TestUploadToFailsWhenParquetFileNotYetWritten (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	1.996s
```
