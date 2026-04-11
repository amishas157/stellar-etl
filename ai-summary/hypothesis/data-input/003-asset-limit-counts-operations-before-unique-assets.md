# H003: Asset export applies `--limit` before deduplicating to unique assets

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Missing asset rows when duplicate operations consume the caller-visible asset limit
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_assets --limit N` should emit up to `N` distinct asset rows, because the command itself deduplicates by `AssetID` before writing. Repeated operations involving the same asset should not consume the caller's unique-asset budget if later operations in-range introduce additional assets.

## Mechanism

`input.GetPaymentOperations()` enforces `limit` on qualifying operations, not on unique transformed assets. `cmd/export_assets` then deduplicates the limited operation slice with `seenIDs`, so if early operations all reference the same asset, the reader can stop before later distinct assets are read and the exporter silently returns fewer unique asset rows than the caller requested.

## Trigger

Run `stellar-etl export_assets --limit 2 --start-ledger <S> --end-ledger <E>` on a range where the first two qualifying payment/manage-sell operations both reference asset `A`, and a later qualifying operation in the same range references distinct asset `B`. The correct output is two rows (`A` and `B`); the current code can emit only `A` because the input reader stops after two operations and the writer drops the duplicate during `seenIDs` deduplication.

## Target Code

- `cmd/export_assets.go:30-34` — obtains a limited slice of qualifying operations from the input package
- `cmd/export_assets.go:39-59` — deduplicates on `seenIDs[transformed.AssetID]` after the input limit has already been applied
- `internal/input/assets.go:21-60` — enforces `limit` on `len(assetSlice)`, i.e. qualifying operations
- `internal/transform/asset.go:14-52` — transforms each operation independently into an `AssetOutput`
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of `assets` to export

## Evidence

The exporter's own logic proves that its real output unit is the unique asset row, not the source operation: it maintains `seenIDs` precisely to collapse repeated asset appearances within one run. Because the limit is consumed before that dedupe step, duplicate operations at the start of the range can starve later unique assets and produce an underfilled-but-plausible export.

## Anti-Evidence

If the first `N` qualifying operations all correspond to distinct assets, the current implementation happens to satisfy the caller's expectation. Users who do not set `--limit` negative avoid the truncation because the full range is read before dedupe.
