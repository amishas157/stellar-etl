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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete LP deposit and withdraw code paths in `internal/transform/operation.go`. Confirmed that `assetA` and `assetB` are declared as `xdr.Asset` (zero-value `Type=0`), only populated inside the `transaction.Result.Successful()` guard, and then unconditionally passed to `addAssetDetailsToOperationDetails()`. The XDR enum maps `AssetType(0)` to `AssetTypeAssetTypeNative`, so `Extract()` returns `assetType="native"` and the helper writes the native sentinel `asset_id=-5706705804583548011`. Critically, upstream Horizon avoids this by declaring `assetA, assetB` as `string` (zero-value `""`), not `xdr.Asset`, so Horizon's failed LP rows carry empty asset strings rather than fabricated `"native"` values. The same bug also affects `LiquidityPoolWithdraw` (lines 1021-1059) which uses the identical pattern.

### Code Paths Examined

- `internal/transform/operation.go:957-984` — LP deposit declares `assetA, assetB xdr.Asset`; only assigned inside `Successful()` block at line 981
- `internal/transform/operation.go:987-989` — `addAssetDetailsToOperationDetails(details, assetA, "reserve_a")` called unconditionally with potentially zero-value asset
- `internal/transform/operation.go:998-999` — Same unconditional call for `assetB` with prefix `"reserve_b"`
- `internal/transform/operation.go:367-386` — `addAssetDetailsToOperationDetails` calls `asset.Extract()`, checks `asset.Type == AssetTypeNative`, sets native sentinel `asset_id`
- `go-stellar-sdk/xdr/xdr_generated.go:1869` — `AssetTypeAssetTypeNative = 0` (the zero value of the enum)
- `go-stellar-sdk/xdr/asset.go:15-21` — `AssetTypeToString[AssetTypeAssetTypeNative] = "native"`
- `go-stellar-sdk/xdr/asset.go:Extract:298-310` — `Extract()` maps `a.Type` directly through `AssetTypeToString`
- `internal/transform/operation.go:1021-1059` — LP withdraw uses identical pattern; same bug
- Horizon `operations_processor.go:600-637` — Horizon declares `assetA, assetB string` (not `xdr.Asset`), avoiding the zero-value native fabrication

### Findings

The bug is confirmed and the mechanism is exactly as described:

1. **Zero-value `xdr.Asset` is semantically `native`**: Go's zero-value for `xdr.Asset` has `Type=0`, which is `AssetTypeAssetTypeNative`. This is a well-known XDR design trait.

2. **Failed LP deposit/withdraw unconditionally exports asset metadata**: The code comment at line 975 says "we will use the defaults (omitted asset and 0 amounts)" — but the asset fields are NOT omitted. Amount fields default to `0` (semantically correct for "no deposit"), but asset fields default to `native` (semantically WRONG for any non-native pool).

3. **Horizon diverges intentionally**: Horizon's equivalent code uses `string` variables (default `""`), not `xdr.Asset`. This means Horizon's failed LP rows carry empty asset strings — effectively "no asset data" — while stellar-etl fabricates `"native"`.

4. **Both deposit AND withdraw are affected**: The `LiquidityPoolWithdraw` case (lines 1021-1059) uses the identical pattern of `xdr.Asset` variables only populated inside `Successful()`, then unconditionally passed to `addAssetDetailsToOperationDetails`.

5. **Downstream impact**: Any analytics query joining on `reserve_a_asset_type` or `reserve_b_asset_type` would incorrectly associate failed LP operations with the native asset. The `asset_id` sentinel further pollutes any asset-dimension joins.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or the LP-specific test file if one exists)
- **Setup**: Construct a `LiquidityPoolDeposit` operation with a non-native asset pair (e.g., `USDC:GABC.../native`). Set `transaction.Result.Successful()` to return `false`.
- **Steps**: Call `extractOperationDetails()` (or `TransformOperation()`) with the failed transaction.
- **Assertion**: Assert that the returned details map does NOT contain `reserve_a_asset_type = "native"` paired with `reserve_b_asset_type = "native"` — or more precisely, that for a failed transaction the asset fields are either absent or reflect the actual pool assets. Currently they will both show `"native"` with `asset_id = -5706705804583548011`. Repeat for `LiquidityPoolWithdraw`.
