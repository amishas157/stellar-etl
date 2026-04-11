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
