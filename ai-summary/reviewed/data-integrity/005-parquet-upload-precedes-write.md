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
