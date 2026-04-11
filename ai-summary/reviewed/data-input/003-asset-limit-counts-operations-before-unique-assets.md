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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `AddArchiveFlags("assets", ...)` call in `export_assets.go:89` generates the flag description "Maximum number of assets to export" (utils/main.go:254). However, the `limit` parameter is passed directly into `GetPaymentOperations()` (assets.go:21) and `GetPaymentOperationsHistoryArchive()` (assets_history_archive.go:13), both of which enforce it on `len(assetSlice)` — the count of qualifying payment/manage-sell-offer operations. The deduplication by `AssetID` happens only afterwards in `export_assets.go:54-56`. This means the limit controls the wrong unit: operations instead of unique assets.

### Code Paths Examined

- `internal/utils/main.go:250-254` — `AddArchiveFlags("assets", ...)` produces help text "Maximum number of assets to export"
- `cmd/export_assets.go:21-34` — extracts `limit` via `MustArchiveFlags` and passes it directly to both asset reader functions
- `internal/input/assets.go:31-57` — `GetPaymentOperations` appends qualifying ops and breaks at `len(assetSlice) >= limit` (operation count, not unique assets)
- `internal/input/assets_history_archive.go:21-47` — `GetPaymentOperationsHistoryArchive` uses identical limit logic on operation count
- `cmd/export_assets.go:40-56` — deduplication loop using `seenIDs[transformed.AssetID]` runs AFTER the limited operation slice is already truncated
- `cmd/export_assets.go:98` — developer comment says "maximum number of operations to export", confirming the mismatch with the flag text

### Findings

The bug is real and analogous to the confirmed success finding 004 (effect limit counts transactions instead of effects), but in the opposite direction:
- **004**: limit counts transactions → oversized effect exports (MORE rows than limit)
- **003**: limit counts operations → underfilled asset exports (FEWER unique assets than limit)

Both `GetPaymentOperations` and `GetPaymentOperationsHistoryArchive` enforce the limit on raw operation count. The `export_assets` command then deduplicates by `AssetID`, potentially discarding duplicates and producing fewer unique rows than the user-specified limit. This is distinct from success finding 002 (asset readers overshoot limit within a ledger), which addresses the per-ledger granularity of the limit check — even if 002's fix were applied, this semantic mismatch would persist.

Severity downgraded from High to Medium because:
- The data that IS exported is individually correct (no field corruption)
- This only triggers when `--limit` is explicitly set AND early operations reference duplicate assets
- Default limit is -1 (no limit), so typical usage is unaffected
- The impact is underfilled exports, not wrong data values

### PoC Guidance

- **Test file**: `internal/input/assets_test.go` or a new PoC test file
- **Setup**: Construct a mock ledger range where early qualifying operations (Payment or ManageSellOffer) reference the same asset, and a later operation references a distinct asset
- **Steps**: Call `GetPaymentOperations(start, end, limit=2, ...)`, then deduplicate the result by AssetID (mirroring export_assets behavior)
- **Assertion**: Assert that the deduplicated result has fewer unique assets than the requested limit of 2, despite additional distinct assets existing later in the range. Compare with calling `GetPaymentOperations` with limit=-1 on the same range and deduplicating — the unlimited call should produce more unique assets.
