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
