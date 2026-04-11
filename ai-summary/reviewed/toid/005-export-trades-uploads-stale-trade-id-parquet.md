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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (success/002 covers `export_operations`, not `export_trades`)

### Trace Summary

Traced `export_trades.go:68-71` confirming `MaybeUpload(parquetPath)` is called before `WriteParquet(transformedTrades, parquetPath, ...)`. Then traced `MaybeUpload` at `command_utils.go:123-146` confirming it immediately dispatches to `GCS.UploadTo()` which opens the on-disk file at call time. `WriteParquet` at `command_utils.go:162-180` is the first function that creates the current run's parquet contents. This is the identical bug pattern confirmed in `export_operations` (success/002) but in a separate command.

### Code Paths Examined

- `cmd/export_trades.go:68-71` — parquet block calls `MaybeUpload` on line 69 then `WriteParquet` on line 70 (upload-before-write confirmed)
- `cmd/command_utils.go:123-146` — `MaybeUpload` immediately invokes `cloudStorage.UploadTo()` which opens the path on disk
- `cmd/command_utils.go:162-180` — `WriteParquet` creates the parquet file writer and writes current run data
- `internal/transform/trade.go:33,119,152` — trade rows derive `HistoryOperationID` and `BuyingOfferID` from TOID-encoded operation IDs
- `cmd/export_assets.go:80-81` — sibling command uses correct write-then-upload order (confirming this is a bug, not design)
- `cmd/export_effects.go:67-68` — another sibling with correct write-then-upload order
- `cmd/export_ledger_entry_changes.go:371-372` — another sibling with correct write-then-upload order

### Findings

The bug is real. Four export commands have the upload-before-write ordering bug: `export_trades`, `export_operations`, `export_ledgers`, and `export_transactions`. Three commands have the correct write-then-upload order: `export_assets`, `export_effects`, and `export_ledger_entry_changes`. The default parquet path from `AddArchiveFlags` is `exported_trades.parquet`, which is reused across runs, making the stale-upload scenario the default behavior.

The trade-specific impact is that TOID-derived fields (`history_operation_id` at trade.go:152, `buying_offer_id` at trade.go:119) in the uploaded parquet will belong to the previous ledger range, not the current one. These are the primary join keys for trade data in downstream BigQuery analytics.

### PoC Guidance

- **Test file**: `cmd/export_trades_poc_test.go` (new file, following pattern from success/002's PoC)
- **Setup**: Read `export_trades.go` source to verify call ordering; create a temp parquet path with stale trade data
- **Steps**:
  1. Assert `MaybeUpload` appears before `WriteParquet` in the source of `export_trades.go` (source ordering test)
  2. Write a stale trade parquet file with `WriteParquet` using ledger-100 TOIDs, then read it back to confirm stale `HistoryOperationID` values
  3. Write a fresh trade parquet file to the same path using ledger-200 TOIDs, read it back, and assert the values differ from the stale file
- **Assertion**: The stale parquet `history_operation_id` differs from the freshly written one, proving upload-before-write sends wrong data
