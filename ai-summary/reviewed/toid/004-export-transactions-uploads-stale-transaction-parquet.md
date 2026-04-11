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
