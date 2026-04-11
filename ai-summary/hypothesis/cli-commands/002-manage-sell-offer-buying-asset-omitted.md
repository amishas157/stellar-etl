# H002: `export_assets` drops the buying side of `ManageSellOffer` asset discovery

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Once `export_assets` chooses to discover assets from `ManageSellOffer` operations, it should preserve both assets carried by that operation. A range where an asset appears only as the `buying` asset of a manage-sell offer should still export that asset, because the offer references both sides on-chain.

## Mechanism

The input readers intentionally admit `ManageSellOffer`, but `TransformAsset()` extracts only `opSellOf.Selling` and ignores `opSellOf.Buying`. As a result, the command can emit the sold asset while silently omitting the bought asset from the same operation. This leaves `history_assets` incomplete for ranges where an asset is only present on the buy side of offer traffic.

## Trigger

Run `export_assets` over a range containing a `ManageSellOffer` whose `buying` asset does not also appear in any plain `Payment` or as the `selling` asset of another manage-sell offer in the same range. The correct output should include both referenced assets; the current export will only include the selling asset.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:42-49` — asset reader explicitly treats `ManageSellOffer` as an asset-discovery source
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:32-39` — history-archive reader does the same
- `internal/transform/asset.go:23-30` — `ManageSellOffer` branch assigns only `opSellOf.Selling`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:26179-26185` — `ManageSellOfferOp` contains both `Selling` and `Buying` assets

## Evidence

The reader already decided that `ManageSellOffer` operations matter for asset exports, so the asymmetry is local to the transform layer: the XDR operation exposes two asset fields, but the exported row is derived from only one of them.

## Anti-Evidence

If the missing buy-side asset later appears in another supported source path, such as a plain `Payment` or as a sell-side asset of another offer, the omission becomes harder to notice because `seenIDs` will let that later row backfill the missing asset.
