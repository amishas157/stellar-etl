# H003: Failed liquidity-pool deposits fabricate `native/native` reserve assets

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a failed `liquidity_pool_deposit` row includes reserve-asset metadata, `reserve_a_*` and `reserve_b_*` should identify the actual asset pair of the referenced pool. If the ETL cannot resolve the pair on failure, those reserve-asset fields should stay absent rather than claiming the pool is `native/native`.

## Mechanism

The deposit branch only assigns `assetA` and `assetB` inside `if transaction.Result.Successful()`. Failed transactions therefore pass zero-value `xdr.Asset{}` values into `addAssetDetailsToOperationDetails()`, and upstream `Asset.Extract()` interprets the zero enum value as `AssetTypeNative`, causing both reserve slots to export as native assets with the native asset ID sentinel.

## Trigger

1. Export a failed `liquidity_pool_deposit` that targets any non-`native/native` pool, for example `USDC:G...` / `native`.
2. Inspect `history_operations.details.reserve_a_asset_type`, `reserve_b_asset_type`, and the companion asset-ID fields.
3. Observe that both reserve assets are exported as `native` even though the referenced pool ID belongs to a different asset pair.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:957-1006` — failed deposit path leaves `assetA` / `assetB` at zero values
- `internal/transform/operation.go:addAssetDetailsToOperationDetails:367-386` — zero-value asset becomes `asset_type = native` and native sentinel asset ID
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/asset.go:Extract:298-342` — `Extract()` maps `a.Type` directly through `AssetTypeToString`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:1869` — zero-value asset type enum is `AssetTypeNative`

## Evidence

The code comment already states that failed deposits keep default omitted assets and zero amounts, but unlike the amount fields the asset metadata is not omitted. Because `assetA` and `assetB` are declared as plain `xdr.Asset` values, their zero-value `Type` is `0`, which upstream defines as native, so the exporter deterministically fabricates `native/native`.

## Anti-Evidence

The row still carries the correct `liquidity_pool_id` and `liquidity_pool_id_strkey`, so a downstream join could recover the true pair from another dataset. That does not change the fact that the reserve asset fields on this row are themselves wrong.
