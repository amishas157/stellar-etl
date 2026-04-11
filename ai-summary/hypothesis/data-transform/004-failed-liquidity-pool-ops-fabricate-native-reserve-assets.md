# H004: Failed liquidity-pool deposit/withdraw rows fabricate native reserve assets

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a `liquidity_pool_deposit` or `liquidity_pool_withdraw` operation fails and the transform cannot derive pool reserve deltas, the reserve asset metadata should remain absent from the detail map. The exporter should not invent `reserve_a_asset_type`, `reserve_a_asset_id`, `reserve_b_asset_type`, or `reserve_b_asset_id` values for reserves it never resolved.

## Mechanism

Both failed LP branches intentionally skip `getLiquidityPoolAndProductDelta()` and leave `assetA` / `assetB` at their Go zero values, while the comment says the failed-path defaults should be "omitted asset and 0 amounts." But `addAssetDetailsToOperationDetails()` passes those zero-value assets into `Asset.Extract()`, and the SDK maps the zero enum value to `"native"`. The transform therefore emits two plausible native reserve assets, plus the native asset ID, for failed LP operations whose reserve composition was never actually resolved.

## Trigger

Export any failed `LIQUIDITY_POOL_DEPOSIT` or `LIQUIDITY_POOL_WITHDRAW` operation. The details row will still include `reserve_a_asset_type = "native"` and `reserve_b_asset_type = "native"` (with the corresponding native asset IDs) even though the success-only pool lookup never ran.

## Target Code

- `internal/transform/operation.go:974-999` — failed deposit path leaves `assetA` / `assetB` unset, then still calls `addAssetDetailsToOperationDetails()`
- `internal/transform/operation.go:1037-1057` — failed withdraw path has the same pattern
- `internal/transform/operation.go:367-384` — helper turns any `xdr.Asset` into exported asset metadata and hard-codes the native asset ID when `asset.Type` is native
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/asset.go:298-343` — zero-value `xdr.Asset` extracts `typ = "native"` without error

## Evidence

The failed-path comment explicitly states the intended fallback is "omitted asset and 0 amounts," but the implementation does not omit the reserve asset fields. Because zero-value `xdr.Asset` is treated as native instead of invalid, the bug produces clean-looking reserve metadata rather than an obvious error.

## Anti-Evidence

Successful LP operations populate the real reserve assets from pool state, so the bug is limited to failed rows. Consumers that discard failed operations entirely will avoid the corruption, but the exported failed-operation details are still wrong whenever those rows are kept for diagnostics or analytics.
