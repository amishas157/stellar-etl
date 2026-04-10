# H001: export_assets Captive-Core Path Zeroes Asset Close Metadata

**Date**: 2026-04-10
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_assets` should emit each asset row with the real ledger close time and ledger sequence from the ledger that contained the triggering payment or manage-sell-offer operation, regardless of whether the export uses datastore or `--use-captive-core`.

## Mechanism

When `commonArgs.UseCaptiveCore` is true, `cmd/export_assets.go` routes the export through `GetPaymentOperationsHistoryArchive()` instead of the normal `GetPaymentOperations()` path. That history-archive reader hard-codes `LedgerCloseMeta` to `xdr.LedgerCloseMeta{}`, but `TransformAsset()` derives `closed_at` and `ledger_sequence` exclusively from that `LedgerCloseMeta`, so the export can silently emit zero-valued metadata (`1970-01-01T00:00:00Z` / ledger sequence `0`) for otherwise valid asset rows.

## Trigger

Run `stellar-etl export_assets --use-captive-core --start-ledger <L> --end-ledger <R>` over any range containing at least one payment or manage-sell-offer operation that introduces an asset. Compare the exported `closed_at` and `ledger_sequence` fields against the same range without `--use-captive-core` or against the source ledger headers.

## Target Code

- `cmd/export_assets.go:30-45` — selects the history-archive reader when `UseCaptiveCore` is true, then feeds its output into `TransformAsset`
- `internal/input/assets_history_archive.go:13-45` — populates `AssetTransformInput` with an empty `xdr.LedgerCloseMeta{}`
- `internal/transform/asset.go:45-50` — derives `ClosedAt` and `LedgerSequence` only from `LedgerCloseMeta`
- `internal/utils/main.go:968-975` — converts `LedgerCloseMeta` into close time and ledger sequence without any alternate source

## Evidence

The normal reader in `internal/input/assets.go` preserves the real `ledger` value in `LedgerCloseMeta`, but the history-archive variant explicitly cannot and stores a zero-value `LedgerCloseMeta` instead. The export path still calls the same transformer, so the metadata fields are sourced from the empty struct rather than from `LedgerSeqNum` or the history-archive ledger header already available in the reader.

## Anti-Evidence

`asset_code`, `asset_issuer`, `asset_type`, and `asset_id` are still extracted directly from the operation body, so only the ledger metadata columns appear corrupted. If downstream consumers ignore `closed_at` and `ledger_sequence`, the bad rows can look superficially valid.
