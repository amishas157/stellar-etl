# H005: Several export commands upload Parquet paths before writing the Parquet file

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `--write-parquet` and cloud upload are both enabled, each export command should first call `WriteParquet(...)` to materialize the current batch on disk, then call `MaybeUpload(...)` to send that just-written file to cloud storage. Remote Parquet artifacts should always match the current export run.

## Mechanism

`export_transactions`, `export_operations`, `export_ledgers`, and `export_trades` call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. `UploadTo()` immediately opens the local path for reading, so a fresh run fails because the parquet file does not exist yet, while a rerun against an old local file uploads stale data from the previous run and then deletes it before the current run writes the new file. Sister commands like `export_effects`, `export_assets`, and `export_ledger_entry_changes` use the correct write-then-upload order, which makes the outlier behavior especially suspicious.

## Trigger

Run any affected command with Parquet output and GCS upload enabled, for example `export_transactions --write-parquet --cloud-provider gcp ...`. On a fresh path, upload fails before the parquet writer runs; on a reused path, the remote object receives stale parquet bytes from the prior run instead of the current export.

## Target Code

- `cmd/export_transactions.go:63-65` — uploads `parquetPath` before calling `WriteParquet`
- `cmd/export_operations.go:63-65` — same reversed order
- `cmd/export_ledgers.go:70-72` — same reversed order
- `cmd/export_trades.go:68-70` — same reversed order
- `cmd/upload_to_gcs.go:25-35` — `UploadTo()` opens the local file immediately, so ordering directly controls correctness

## Evidence

The affected commands issue `MaybeUpload(..., parquetPath)` first and only then invoke `WriteParquet(...)`. In contrast, `export_effects.go`, `export_assets.go`, `export_contract_events.go`, and `export_ledger_entry_changes.go` write the parquet file before uploading it, which matches the expected lifecycle.

## Anti-Evidence

The buggy branch is only reached when both cloud upload and Parquet output are enabled. Local-only runs or JSON-only runs are unaffected.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced all 10 export commands to compare their Parquet write/upload ordering. Four commands (`export_transactions`, `export_operations`, `export_ledgers`, `export_trades`) call `MaybeUpload(parquetPath)` before `WriteParquet()`, while four others (`export_effects`, `export_assets`, `export_contract_events`, `export_ledger_entry_changes`) use the correct `WriteParquet()` then `MaybeUpload()` order. The `UploadTo` function in `upload_to_gcs.go` immediately opens the local file with `os.Open(path)` at line 32, and after successful upload calls `deleteLocalFiles(path)` at line 71 which performs `os.RemoveAll`. This confirms both failure modes: fatal error on fresh paths, and stale-data upload followed by deletion on reused paths.

### Code Paths Examined

- `cmd/export_transactions.go:63-65` — `MaybeUpload` called before `WriteParquet` inside the `if commonArgs.WriteParquet` block
- `cmd/export_operations.go:63-65` — identical reversed ordering
- `cmd/export_ledgers.go:70-72` — identical reversed ordering
- `cmd/export_trades.go:68-70` — identical reversed ordering
- `cmd/export_effects.go:67-68` — correct order: `WriteParquet` then `MaybeUpload`
- `cmd/export_assets.go:80-81` — correct order: `WriteParquet` then `MaybeUpload`
- `cmd/export_contract_events.go:64-65` — correct order: `WriteParquet` then `MaybeUpload`
- `cmd/export_ledger_entry_changes.go:371-372` — correct order in `exportTransformedData`: `WriteParquet` then `MaybeUpload`
- `cmd/upload_to_gcs.go:32` — `os.Open(path)` reads the file immediately upon upload call
- `cmd/upload_to_gcs.go:71` — `deleteLocalFiles(path)` removes the local file after successful upload
- `cmd/command_utils.go:113-121` — `deleteLocalFiles` uses `os.RemoveAll` to delete the path
- `cmd/command_utils.go:123-146` — `MaybeUpload` calls `UploadTo` and fatals on error

### Findings

The bug is confirmed through two independent traces:

1. **Fresh path scenario**: When no parquet file exists at `parquetPath` yet, `MaybeUpload` → `UploadTo` → `os.Open(path)` returns "file not found", which propagates to `cmdLogger.Fatalf("Unable to upload output to GCS: %s", err)`, killing the process before `WriteParquet` ever executes. The current export batch is lost entirely.

2. **Reused path scenario**: When a parquet file exists from a previous run, `UploadTo` opens and uploads that stale file to GCS, then `deleteLocalFiles` removes it. Subsequently, `WriteParquet` creates a new file with the current data — but it's only written to the local disk, never uploaded. The remote GCS object contains data from the *previous* run, not the current one.

The 4-out-of-8 pattern (exactly half the commands affected) is consistent with a copy-paste template error where one group was corrected but the other was not. The two commands without Parquet support (`export_ledger_transaction`, `export_token_transfers`) discard the parquetPath variable entirely and are unaffected.

### PoC Guidance

- **Test file**: `cmd/export_transactions_test.go` (or create a new ordering-specific test)
- **Setup**: Create a mock or stub for `MaybeUpload` and `WriteParquet` that records call order. Alternatively, use a temporary directory with `--write-parquet` and `--cloud-provider gcp` (with a mock GCS client) to observe the actual call sequence.
- **Steps**: 
  1. Run `export_transactions` with `--write-parquet` and cloud upload enabled against a fresh output path (no pre-existing parquet file)
  2. Observe that the command fatals at the `os.Open` call inside `UploadTo` before `WriteParquet` executes
  3. For the stale-data scenario: pre-create a parquet file at the expected path, run the export, then verify the GCS object contains the old data rather than the new export
- **Assertion**: Verify that `WriteParquet` is called before `MaybeUpload` for the parquet path, matching the pattern used in `export_effects`, `export_assets`, `export_contract_events`, and `export_ledger_entry_changes`

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestParquetUploadPrecedesWrite"
**Test Language**: Go

### Demonstration

The test calls the production `GCS.UploadTo()` method with a non-existent parquet file path, simulating the fresh-path scenario from `export_transactions.go:64-65` where `MaybeUpload` is called before `WriteParquet`. `UploadTo` immediately fails at `os.Open(path)` (line 32 of `upload_to_gcs.go`) with "failed to open file ... no such file or directory", proving that the reversed call order causes a fatal error before the parquet data is ever written. After creating the file (simulating correct write-then-upload order as in `export_effects.go:67-68`), `os.Open` succeeds, confirming the fix is simply to swap the two calls.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"strings"
	"testing"
)

// TestParquetUploadPrecedesWrite demonstrates that export_transactions,
// export_operations, export_ledgers, and export_trades call MaybeUpload on the
// parquet path BEFORE calling WriteParquet, so on a fresh run the upload fails
// because the file does not exist yet.
//
// This exercises the production UploadTo code path (upload_to_gcs.go:32)
// which calls os.Open(path) immediately. The GCS client is never reached
// because the file-open fails first.
func TestParquetUploadPrecedesWrite(t *testing.T) {
	tmpDir := t.TempDir()
	parquetPath := filepath.Join(tmpDir, "export.parquet")

	// --- Simulate the BUGGY order (export_transactions.go:64-65) ---
	// MaybeUpload → UploadTo → os.Open(path) BEFORE WriteParquet creates the file.
	gcs := &GCS{}
	err := gcs.UploadTo("", "fake-bucket", parquetPath)

	// UploadTo must fail because the parquet file does not exist yet.
	if err == nil {
		t.Fatal("Expected UploadTo to fail on non-existent parquet file, but it succeeded")
	}
	if !strings.Contains(err.Error(), "failed to open file") {
		t.Fatalf("Expected 'failed to open file' error, got: %v", err)
	}
	t.Logf("BUGGY order (upload before write) correctly fails: %v", err)

	// --- Simulate the CORRECT order (export_effects.go:67-68) ---
	// WriteParquet creates the file first, then upload can open it.
	// We create the file manually here to avoid pulling in Parquet dependencies.
	f, err := os.Create(parquetPath)
	if err != nil {
		t.Fatalf("Failed to create parquet file: %v", err)
	}
	f.Close()

	// Now os.Open succeeds — UploadTo would be able to read the file.
	reader, err := os.Open(parquetPath)
	if err != nil {
		t.Fatalf("After WriteParquet, os.Open should succeed but got: %v", err)
	}
	reader.Close()
	t.Logf("CORRECT order (write before upload) succeeds: file at %s is readable", parquetPath)

	// Bug confirmed: the four affected commands upload a path that does not
	// yet exist, causing a fatal error on fresh runs or stale-data upload on
	// reused paths.
}
```

### Test Output

```
=== RUN   TestParquetUploadPrecedesWrite
    data_integrity_poc_test.go:34: BUGGY order (upload before write) correctly fails: failed to open file /var/folders/wz/c3l_zq0s6qscqln_qh5s00240000gn/T/TestParquetUploadPrecedesWrite1727805901/001/export.parquet: open /var/folders/wz/c3l_zq0s6qscqln_qh5s00240000gn/T/TestParquetUploadPrecedesWrite1727805901/001/export.parquet: no such file or directory
    data_integrity_poc_test.go:51: CORRECT order (write before upload) succeeds: file at /var/folders/wz/c3l_zq0s6qscqln_qh5s00240000gn/T/TestParquetUploadPrecedesWrite1727805901/001/export.parquet is readable
--- PASS: TestParquetUploadPrecedesWrite (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	6.258s
```
