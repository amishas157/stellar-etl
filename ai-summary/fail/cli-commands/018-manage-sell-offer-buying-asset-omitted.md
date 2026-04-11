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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantially equivalent to 017-path-payment-assets-omitted
**Failed At**: reviewer

### Trace Summary

Traced the complete `export_assets` pipeline: `cmd/export_assets.go` calls `TransformAsset()` once per admitted operation and expects a single `AssetOutput` return value. The function signature (`AssetOutput, error`) is structurally designed for one-asset-per-operation extraction. For `ManageSellOffer`, line 29 assigns `asset = opSellOf.Selling`, ignoring `Buying`. This is consistent with the intentionally limited scope established by the 017 review.

### Code Paths Examined

- `cmd/export_assets.go:44-50` — caller iterates operations, calls `TransformAsset` once per op, expects single `AssetOutput`
- `internal/transform/asset.go:14` — function signature returns `(AssetOutput, error)`, not a slice — structurally one-asset-per-call
- `internal/transform/asset.go:24-29` — `ManageSellOffer` branch: `asset = opSellOf.Selling`, `Buying` not referenced
- `internal/input/assets.go:42` — input filter admits `ManageSellOffer` by type, passes full operation including both asset fields
- `internal/input/assets_history_archive.go:32` — identical filter in archive-backed reader

### Why It Failed

This is **working-as-designed behavior** and substantially equivalent to the previously reviewed hypothesis 017-path-payment-assets-omitted. The same reasoning applies:

1. **Structural design choice**: `TransformAsset` returns a single `AssetOutput` per call (not a slice). The caller in `cmd/export_assets.go:44-50` processes one result per operation. This one-asset-per-operation design is a deliberate architectural choice, not an oversight.

2. **Consistent limited scope**: The 017 review established that `export_assets` has an intentionally limited scope for asset discovery. The command extracts the *primary* asset from each admitted operation type: `Payment.Asset` (the transferred asset) and `ManageSellOffer.Selling` (the offered asset). Extracting only the Selling side is consistent with this limited extraction pattern.

3. **Same class of limitation**: If extracting only Selling from ManageSellOffer is a bug, then by the same logic: not extracting from `ManageBuyOffer`, `CreatePassiveSellOffer`, `ChangeTrust`, and all path-payment types would also be bugs. The 017 review already rejected this reasoning — the command was never designed for comprehensive asset discovery.

4. **No documented contract violation**: The command description says "Exports the assets that are created from payment operations." The code does exactly what it documents — a limited-scope asset reference export.

### Lesson Learned

Within an intentionally limited-scope command, extracting a subset of fields from an admitted operation type is part of the same design philosophy as filtering operation types in the first place. A hypothesis about incomplete extraction within an admitted type is essentially a variant of a hypothesis about incomplete type coverage — both describe the same intentional limitation at different granularities.
