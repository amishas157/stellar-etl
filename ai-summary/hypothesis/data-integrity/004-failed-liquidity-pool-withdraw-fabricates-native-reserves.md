# H004: Failed liquidity-pool withdrawals fabricate `native/native` reserve assets

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a failed `liquidity_pool_withdraw` row exports reserve-asset metadata, `reserve_a_*` and `reserve_b_*` should still identify the actual pool assets, or else those fields should remain unset. A failed withdrawal against a credit/native pool should never serialize both reserve assets as native.

## Mechanism

The withdraw branch also initializes `assetA` and `assetB` as zero-value `xdr.Asset` structs and only populates them inside the success-only block. On failure it still calls `addAssetDetailsToOperationDetails()` for both reserve slots, and the zero enum value again maps to `AssetTypeNative`, so the ETL emits a believable but false `native/native` pair.

## Trigger

1. Export a failed `liquidity_pool_withdraw` against any pool whose reserves are not both native.
2. Inspect `history_operations.details.reserve_a_asset_type`, `reserve_b_asset_type`, and the reserve asset IDs.
3. Observe that both reserve assets are exported as native even though the pool identified by `liquidity_pool_id` uses a different pair.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:1022-1061` — failed withdraw path leaves `assetA` / `assetB` at zero values
- `internal/transform/operation.go:addAssetDetailsToOperationDetails:367-386` — writes native asset metadata for zero-value assets
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/asset.go:Extract:298-342` — zero-value asset extraction behavior
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:1869` — `AssetTypeNative` is enum value `0`

## Evidence

The only place the withdraw branch learns the pool assets is the success-only `getLiquidityPoolAndProductDelta()` block. When that block is skipped, both reserve asset structs stay zero-valued but are still serialized through the generic asset helper, which deterministically reports them as native assets.

## Anti-Evidence

As with failed deposits, the pool identifier fields remain correct and could let a downstream consumer reconstruct the real asset pair from another export. The immediate withdrawal row still contains wrong reserve-asset metadata, which is enough for silent structural corruption in any consumer that trusts the row at face value.
