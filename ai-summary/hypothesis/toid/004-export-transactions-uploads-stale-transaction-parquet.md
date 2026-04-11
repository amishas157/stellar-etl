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
