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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced both the stellar-etl `extractOperationDetails()` LP deposit path (lines 960–1019) and LP withdraw path (lines 1021–1060) in `internal/transform/operation.go`. Confirmed that zero-value `xdr.Asset` variables are passed to `addAssetDetailsToOperationDetails()` when `transaction.Result.Successful()` is false, and that `Extract()` maps the zero enum value (`AssetTypeAssetTypeNative = 0`) to `"native"`. Then traced the canonical upstream implementation in `stellar/go` SDK at `processors/operation/liquidity_pool_deposit_details.go` and `liquidity_pool_withdraw_details.go`. The upstream SDK uses the **identical** pattern with the **identical** comment ("we will use the defaults (omitted asset and 0 amounts) if the transaction failed") and the same zero-value `xdr.Asset` → `Extract()` → `"native"` path.

### Code Paths Examined

- `internal/transform/operation.go:960-1019` — LP deposit: declares `assetA, assetB xdr.Asset` (zero values), populates only on `Successful()`, then unconditionally calls `addAssetDetailsToOperationDetails()` with zero-value assets
- `internal/transform/operation.go:1021-1060` — LP withdraw: identical pattern
- `internal/transform/operation.go:367-387` — `addAssetDetailsToOperationDetails()`: calls `asset.Extract()`, checks `asset.Type == AssetTypeAssetTypeNative`, sets `asset_type = "native"` and hardcoded `asset_id`
- `stellar/go xdr/xdr_generated.go:1869` — `AssetTypeAssetTypeNative = 0` (the Go zero value for the enum)
- `stellar/go xdr/asset.go:298-318` — `Extract()` maps zero-value `AssetType` to `"native"` string via `AssetTypeToString` lookup
- `stellar/go processors/operation/liquidity_pool_deposit_details.go:68-100` — upstream SDK: identical pattern, identical comment, same zero-value `xdr.Asset` → `Extract()` → `"native"`
- `stellar/go processors/operation/liquidity_pool_withdraw_details.go:51-83` — upstream SDK: identical pattern for withdrawals

### Why It Failed

The behavior is **working as designed** and matches the canonical Horizon SDK implementation exactly. The upstream `stellar/go` SDK (`processors/operation/liquidity_pool_deposit_details.go` and `liquidity_pool_withdraw_details.go`) uses the identical pattern: declare `assetA, assetB xdr.Asset` at zero values, populate only on `o.Transaction.Successful()`, then unconditionally call `Extract()` outside the success block. Both codebases share the same comment acknowledging this behavior. While the comment says "omitted asset," neither codebase actually omits the field — both emit `"native"` from the zero-value enum. This is the ecosystem convention for failed LP operations, not a bug introduced by stellar-etl. Per the project's scope rules, bugs in the upstream `stellar/go` SDK should be reported to the SDK repo, not flagged as ETL data corruption.

### Lesson Learned

This is the same pattern as the failed path-payment hypothesis (fail/030): when the upstream SDK uses an identical zero-default pattern for failed operations — including the same misleading comment — the ETL is correctly replicating the ecosystem convention. The "omitted asset" comment is misleading in both codebases, but the actual behavior is consistent across the ecosystem.
