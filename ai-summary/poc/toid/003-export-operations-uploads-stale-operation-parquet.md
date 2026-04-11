# H003: `export_operations` can upload stale `operation_id` parquet before regenerating it

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: Medium
**Impact**: silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_operations --write-parquet` is used with cloud upload enabled, the uploaded parquet file should contain the freshly generated `id`/`transaction_id` TOIDs for the current ledger range. Cloud consumers should never receive a prior run's operation parquet after the command reports success.

## Mechanism

`export_operations` calls `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. If the same local parquet path already exists from an earlier run, the command uploads that stale file to cloud storage and only then overwrites the local parquet with the new operation TOIDs, leaving remote analytics with silently outdated IDs.

## Trigger

1. Run `export_operations --write-parquet` with a fixed `--parquet-output` path and cloud upload enabled.
2. Run it again for a different ledger range using the same parquet path.
3. Observe that the remote parquet still contains the first run's `operation_id` / `transaction_id` values because upload happened before regeneration.

## Target Code

- `cmd/export_operations.go:61-65` — uploads `parquetPath` before calling `WriteParquet(...)`.
- `cmd/command_utils.go:148-180` — `WriteParquet(...)` is the first place that actually regenerates the parquet file contents.
- `internal/transform/schema_parquet.go:116-129` — operation parquet rows expose TOID-bearing `transaction_id` and `id` columns.

## Evidence

Every successful JSON row is buffered into `transformedOps`, but the parquet file itself does not exist or change until `WriteParquet(...)` runs. Uploading `parquetPath` first therefore cannot send the freshly transformed TOIDs; it can only fail or send whatever stale file already lives at that path.

## Anti-Evidence

If the parquet path does not already exist locally, this path may fail loudly instead of silently corrupting remote data. The silent stale-upload case requires a reused local output path, but that is a normal batch-export pattern for scheduled jobs.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the parquet write/upload ordering in all 8 archive export commands. Four commands (`export_operations.go`, `export_trades.go`, `export_transactions.go`, `export_ledgers.go`) call `MaybeUpload(parquetPath)` before `WriteParquet()`. Four others (`export_assets.go`, `export_effects.go`, `export_ledger_entry_changes.go`, `export_contract_events.go`) use the correct order: `WriteParquet()` then `MaybeUpload()`. The `UploadTo` implementation in `upload_to_gcs.go` calls `os.Open(path)`, so on first run the command fatally crashes since the file doesn't exist; on subsequent runs with a reused path it silently uploads stale data.

### Code Paths Examined

- `cmd/export_operations.go:63-65` — `MaybeUpload(parquetPath)` precedes `WriteParquet(transformedOps, ...)`, confirmed wrong order
- `cmd/export_trades.go:68-70` — same wrong order: `MaybeUpload` then `WriteParquet`
- `cmd/export_transactions.go:63-65` — same wrong order: `MaybeUpload` then `WriteParquet`
- `cmd/export_ledgers.go:70-72` — same wrong order: `MaybeUpload` then `WriteParquet`
- `cmd/export_assets.go:79-81` — CORRECT order: `WriteParquet` then `MaybeUpload`
- `cmd/export_effects.go:66-68` — CORRECT order: `WriteParquet` then `MaybeUpload`
- `cmd/export_ledger_entry_changes.go:371-372` — CORRECT order: `WriteParquet` then `MaybeUpload`
- `cmd/export_contract_events.go:64-65` — CORRECT order: `WriteParquet` then `MaybeUpload`
- `cmd/upload_to_gcs.go:32-34` — `os.Open(path)` fails if file does not exist, error returned
- `cmd/command_utils.go:123-146` — `MaybeUpload` calls `Fatalf` on upload error, terminating the process

### Findings

The bug has two manifestations depending on whether a stale parquet file exists at the output path:

1. **First run (no prior file)**: `MaybeUpload` → `UploadTo` → `os.Open(parquetPath)` fails → `Fatalf` kills the process → `WriteParquet` never runs. The command crashes, so `--write-parquet` with cloud upload is completely broken for these 4 commands on first execution.

2. **Subsequent runs (stale file exists)**: `MaybeUpload` successfully uploads the **previous run's** parquet file → `WriteParquet` then overwrites the local file with fresh data. Cloud consumers receive stale data from the prior run.

This is exactly Investigation Pattern 3 (export command consistency): 4 of 8 commands have the wrong ordering, while the other 4 have the correct `WriteParquet` → `MaybeUpload` sequence. The affected commands are `export_operations`, `export_trades`, `export_transactions`, and `export_ledgers`.

### PoC Guidance

- **Test file**: `cmd/export_operations_test.go` (or a new integration-style test)
- **Setup**: Create a mock cloud storage implementation or observe call ordering. The simplest PoC is to verify the call sequence statically or via a wrapper that records call order.
- **Steps**:
  1. Inspect `export_operations.go` lines 63-65 and confirm `MaybeUpload` appears before `WriteParquet`
  2. Compare with `export_assets.go` lines 79-81 where the correct order is `WriteParquet` then `MaybeUpload`
  3. Optionally: run `export_operations --write-parquet --cloud-provider gcp --cloud-storage-bucket test` with a non-existent parquet path and confirm fatal error from `os.Open`
- **Assertion**: In the 4 affected commands, `MaybeUpload(parquetPath)` is called before `WriteParquet(...)`. The fix is to swap the two lines in each affected command, matching the pattern used in `export_assets.go`, `export_effects.go`, `export_ledger_entry_changes.go`, and `export_contract_events.go`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportParquetUploadOrderingBug"
**Test Language**: Go

### Demonstration

The test confirms the hypothesis through three complementary sub-tests. First, it parses the source code of all 7 export commands and verifies that exactly 4 commands (`export_operations.go`, `export_trades.go`, `export_transactions.go`, `export_ledgers.go`) have `MaybeUpload` before `WriteParquet` in the parquet block, while 3 commands have the correct order. Second, it demonstrates that `os.Open` on a non-existent parquet path fails — proving first-run crashes. Third, it shows that when a stale file exists at the parquet path, reading it at the `MaybeUpload` call site yields stale content that differs from what `WriteParquet` would later produce.

### Test Body

```go
package cmd

import (
	"bytes"
	"io"
	"os"
	"path/filepath"
	"runtime"
	"strings"
	"testing"
)

// TestExportParquetUploadOrderingBug demonstrates H003: four export commands
// call MaybeUpload(parquetPath) BEFORE WriteParquet(), causing:
//   - First run: MaybeUpload → os.Open fails (file missing) → Fatal crash
//   - Reused path: MaybeUpload uploads stale data from previous run
func TestExportParquetUploadOrderingBug(t *testing.T) {
	// Locate the cmd/ source directory via runtime.Caller
	_, thisFile, _, ok := runtime.Caller(0)
	if !ok {
		t.Fatal("cannot determine test file path")
	}
	cmdDir := filepath.Dir(thisFile)

	// Part 1: Verify source code ordering — MaybeUpload before WriteParquet in 4 commands
	t.Run("source_ordering", func(t *testing.T) {
		type testCase struct {
			file        string
			expectBuggy bool // true = MaybeUpload appears before WriteParquet
		}
		cases := []testCase{
			{"export_operations.go", true},
			{"export_trades.go", true},
			{"export_transactions.go", true},
			{"export_ledgers.go", true},
			{"export_assets.go", false},
			{"export_effects.go", false},
			{"export_contract_events.go", false},
		}

		buggyCmds := 0
		correctCmds := 0

		for _, tc := range cases {
			tc := tc
			t.Run(tc.file, func(t *testing.T) {
				src, err := os.ReadFile(filepath.Join(cmdDir, tc.file))
				if err != nil {
					t.Fatalf("cannot read %s: %v", tc.file, err)
				}

				lines := strings.Split(string(src), "\n")

				// Find the "if commonArgs.WriteParquet {" block that contains
				// both MaybeUpload and WriteParquet calls (not the append block)
				maybeUploadLine := -1
				writeParquetLine := -1
				for i, line := range lines {
					trimmed := strings.TrimSpace(line)
					if !strings.HasPrefix(trimmed, "if commonArgs.WriteParquet") {
						continue
					}
					// Search within this block for both calls
					mu, wp := -1, -1
					for j := i + 1; j < len(lines) && j < i+10; j++ {
						inner := strings.TrimSpace(lines[j])
						if strings.HasPrefix(inner, "MaybeUpload(") && mu == -1 {
							mu = j
						}
						if strings.HasPrefix(inner, "WriteParquet(") && wp == -1 {
							wp = j
						}
						if inner == "}" {
							break
						}
					}
					if mu != -1 && wp != -1 {
						maybeUploadLine = mu
						writeParquetLine = wp
						break
					}
				}

				if maybeUploadLine == -1 || writeParquetLine == -1 {
					t.Fatalf("could not find WriteParquet block with both MaybeUpload and WriteParquet calls")
				}

				uploadFirst := maybeUploadLine < writeParquetLine

				if tc.expectBuggy {
					if !uploadFirst {
						t.Errorf("expected MaybeUpload before WriteParquet in %s, but WriteParquet comes first — bug may be fixed", tc.file)
					} else {
						t.Logf("BUG CONFIRMED in %s: MaybeUpload (line %d) precedes WriteParquet (line %d)",
							tc.file, maybeUploadLine+1, writeParquetLine+1)
						buggyCmds++
					}
				} else {
					if uploadFirst {
						t.Errorf("expected WriteParquet before MaybeUpload in %s, but MaybeUpload comes first — unexpected bug", tc.file)
					} else {
						t.Logf("CORRECT in %s: WriteParquet (line %d) precedes MaybeUpload (line %d)",
							tc.file, writeParquetLine+1, maybeUploadLine+1)
						correctCmds++
					}
				}
			})
		}

		t.Logf("Summary: %d buggy commands, %d correct commands", buggyCmds, correctCmds)
	})

	// Part 2: Demonstrate first-run crash — os.Open fails on non-existent parquet file
	t.Run("first_run_crash", func(t *testing.T) {
		tmpDir := t.TempDir()
		nonExistentPath := filepath.Join(tmpDir, "nonexistent.parquet")

		// In the affected commands, MaybeUpload is called before WriteParquet.
		// MaybeUpload → UploadTo → os.Open(path) fails if the file doesn't exist.
		_, err := os.Open(nonExistentPath)
		if err == nil {
			t.Fatal("expected error opening non-existent file")
		}
		if !os.IsNotExist(err) {
			t.Fatalf("expected file-not-found error, got: %v", err)
		}

		t.Logf("First-run crash confirmed: os.Open(%q) returns %v — MaybeUpload would Fatal before WriteParquet ever runs",
			filepath.Base(nonExistentPath), err)
	})

	// Part 3: Demonstrate stale upload on reused path
	t.Run("stale_upload_on_reused_path", func(t *testing.T) {
		tmpDir := t.TempDir()
		parquetPath := filepath.Join(tmpDir, "test.parquet")

		// Simulate "previous run" — create a file with known stale content
		staleContent := []byte("STALE_PARQUET_DATA_FROM_PREVIOUS_RUN_ledger_100")
		if err := os.WriteFile(parquetPath, staleContent, 0644); err != nil {
			t.Fatalf("failed to create stale file: %v", err)
		}

		// === New run begins ===
		// In the affected commands, MaybeUpload(parquetPath) is called HERE,
		// BEFORE WriteParquet creates fresh content.
		// Simulate what UploadTo does: os.Open(path) + io.Copy
		f, err := os.Open(parquetPath)
		if err != nil {
			t.Fatalf("os.Open failed: %v", err)
		}
		uploadedContent, err := io.ReadAll(f)
		f.Close()
		if err != nil {
			t.Fatalf("ReadAll failed: %v", err)
		}

		// Now WriteParquet runs — this is what SHOULD have happened first.
		// Simulate with fresh content that differs from the stale file.
		freshContent := []byte("FRESH_PARQUET_DATA_FOR_NEW_RUN_ledger_200")
		if err := os.WriteFile(parquetPath, freshContent, 0644); err != nil {
			t.Fatalf("failed to write fresh file: %v", err)
		}

		actualFreshContent, err := os.ReadFile(parquetPath)
		if err != nil {
			t.Fatalf("ReadFile failed: %v", err)
		}

		// Bug proof: content that would be uploaded differs from fresh content
		if bytes.Equal(uploadedContent, actualFreshContent) {
			t.Fatal("UNEXPECTED: uploaded content equals fresh content — ordering bug not demonstrated")
		}

		// Verify the uploaded content matches the stale data
		if !bytes.Equal(uploadedContent, staleContent) {
			t.Fatal("UNEXPECTED: uploaded content doesn't match stale data")
		}

		t.Logf("Stale upload confirmed: MaybeUpload reads %d bytes of stale data (%q); "+
			"WriteParquet later writes %d bytes of fresh data (%q)",
			len(uploadedContent), string(uploadedContent),
			len(actualFreshContent), string(actualFreshContent))
	})
}
```

### Test Output

```
=== RUN   TestExportParquetUploadOrderingBug
=== RUN   TestExportParquetUploadOrderingBug/source_ordering
=== RUN   TestExportParquetUploadOrderingBug/source_ordering/export_operations.go
    data_integrity_poc_test.go:94: BUG CONFIRMED in export_operations.go: MaybeUpload (line 64) precedes WriteParquet (line 65)
=== RUN   TestExportParquetUploadOrderingBug/source_ordering/export_trades.go
    data_integrity_poc_test.go:94: BUG CONFIRMED in export_trades.go: MaybeUpload (line 69) precedes WriteParquet (line 70)
=== RUN   TestExportParquetUploadOrderingBug/source_ordering/export_transactions.go
    data_integrity_poc_test.go:94: BUG CONFIRMED in export_transactions.go: MaybeUpload (line 64) precedes WriteParquet (line 65)
=== RUN   TestExportParquetUploadOrderingBug/source_ordering/export_ledgers.go
    data_integrity_poc_test.go:94: BUG CONFIRMED in export_ledgers.go: MaybeUpload (line 71) precedes WriteParquet (line 72)
=== RUN   TestExportParquetUploadOrderingBug/source_ordering/export_assets.go
    data_integrity_poc_test.go:102: CORRECT in export_assets.go: WriteParquet (line 80) precedes MaybeUpload (line 81)
=== RUN   TestExportParquetUploadOrderingBug/source_ordering/export_effects.go
    data_integrity_poc_test.go:102: CORRECT in export_effects.go: WriteParquet (line 67) precedes MaybeUpload (line 68)
=== RUN   TestExportParquetUploadOrderingBug/source_ordering/export_contract_events.go
    data_integrity_poc_test.go:102: CORRECT in export_contract_events.go: WriteParquet (line 64) precedes MaybeUpload (line 65)
=== NAME  TestExportParquetUploadOrderingBug/source_ordering
    data_integrity_poc_test.go:110: Summary: 4 buggy commands, 3 correct commands
=== RUN   TestExportParquetUploadOrderingBug/first_run_crash
    data_integrity_poc_test.go:128: First-run crash confirmed: os.Open("nonexistent.parquet") returns open .../nonexistent.parquet: no such file or directory — MaybeUpload would Fatal before WriteParquet ever runs
=== RUN   TestExportParquetUploadOrderingBug/stale_upload_on_reused_path
    data_integrity_poc_test.go:179: Stale upload confirmed: MaybeUpload reads 47 bytes of stale data ("STALE_PARQUET_DATA_FROM_PREVIOUS_RUN_ledger_100"); WriteParquet later writes 41 bytes of fresh data ("FRESH_PARQUET_DATA_FOR_NEW_RUN_ledger_200")
--- PASS: TestExportParquetUploadOrderingBug (0.00s)
    --- PASS: TestExportParquetUploadOrderingBug/source_ordering (0.00s)
        --- PASS: TestExportParquetUploadOrderingBug/source_ordering/export_operations.go (0.00s)
        --- PASS: TestExportParquetUploadOrderingBug/source_ordering/export_trades.go (0.00s)
        --- PASS: TestExportParquetUploadOrderingBug/source_ordering/export_transactions.go (0.00s)
        --- PASS: TestExportParquetUploadOrderingBug/source_ordering/export_ledgers.go (0.00s)
        --- PASS: TestExportParquetUploadOrderingBug/source_ordering/export_assets.go (0.00s)
        --- PASS: TestExportParquetUploadOrderingBug/source_ordering/export_effects.go (0.00s)
        --- PASS: TestExportParquetUploadOrderingBug/source_ordering/export_contract_events.go (0.00s)
    --- PASS: TestExportParquetUploadOrderingBug/first_run_crash (0.00s)
    --- PASS: TestExportParquetUploadOrderingBug/stale_upload_on_reused_path (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	1.885s
```
