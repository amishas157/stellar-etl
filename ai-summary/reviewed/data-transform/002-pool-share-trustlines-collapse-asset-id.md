# H002: Pool-share trustlines collapse distinct pools into one `asset_id`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Two pool-share trustlines with different `liquidity_pool_id` values should export different `asset_id` values, because the pool ID is part of the pool-share asset identity on chain. Distinct liquidity pools should not collapse onto a single surrogate asset key.

## Mechanism

For pool-share trustlines, `TransformTrustline()` fills `LiquidityPoolID` and hard-codes `asset_type = "pool_share"`, but it hashes `asset_id` from only `outputAssetCode`, `outputAssetIssuer`, and `asset.Type.String()`. Because pool-share trustlines have no code or issuer and the hash input never includes `LiquidityPoolId`, every pool-share trustline produces the same `asset_id` regardless of which pool the shares belong to.

## Trigger

Export at least two trustline rows whose assets are pool shares for different liquidity pools. The emitted rows will show different `liquidity_pool_id` / `liquidity_pool_id_strkey` values but the same `asset_id`, proving the hash collapsed distinct pool-share assets.

## Target Code

- `internal/transform/trustline.go:43-58` — pool-share trustlines set `LiquidityPoolID` but compute `asset_id` without the pool ID
- `internal/transform/asset.go:72-76` — `FarmHashAsset()` hashes only `assetCode + assetIssuer + assetType`
- `go-stellar-sdk/xdr/trust_line_asset.go:58-80` — trustline asset encoding for pool shares includes `LiquidityPoolId`
- `go-stellar-sdk/xdr/trust_line_asset.go:86-98` — pool-share asset equality is keyed by `LiquidityPoolId`

## Evidence

The XDR helper code makes clear that pool-share trustline assets are distinguished by `LiquidityPoolId`, not merely by a generic pool-share type tag. But the ETL's `asset_id` hash for trustlines excludes that ID entirely, so all pool-share trustlines are forced into one hash bucket.

## Anti-Evidence

The trustline row also exports `liquidity_pool_id` and `liquidity_pool_id_strkey`, so consumers who always key pool-share analytics off those columns can recover the distinction. However, the exported `asset_id` column itself is still wrong and cannot safely serve as a cross-row asset key for pool-share trustlines.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `TransformTrustline()` in `trustline.go:18-91` end-to-end. For pool-share assets (line 43-55), `outputAssetCode` and `outputAssetIssuer` remain their zero values (empty strings) from line 34. Only `poolID`, `poolIDStrkey`, and `assetType` are set. Line 57 then calls `FarmHashAsset("", "", asset.Type.String())`, which always produces the same hash for every pool-share trustline regardless of which liquidity pool it references. The existing test (`trustline_test.go:149-166`) only includes one pool-share test case, so the collision was never caught. A second pool-share trustline with a different `LiquidityPoolId` would produce the identical `AssetID: -1967220342708457407`.

### Code Paths Examined

- `internal/transform/trustline.go:34` — `var assetType, outputAssetCode, outputAssetIssuer, poolID, poolIDStrkey string` declares all as empty strings
- `internal/transform/trustline.go:43-55` — pool-share branch sets `poolIDStrkey`, `poolID`, `assetType="pool_share"` but never sets `outputAssetCode` or `outputAssetIssuer`
- `internal/transform/trustline.go:57` — `FarmHashAsset(outputAssetCode, outputAssetIssuer, asset.Type.String())` called with `("", "", "AssetTypePoolShare")` for all pool-share trustlines
- `internal/transform/asset.go:72-76` — `FarmHashAsset` concatenates all three arguments and hashes; identical inputs always produce identical output
- `internal/transform/trustline_test.go:149-166` — only one pool-share test case exists with `AssetID: -1967220342708457407`; a second pool-share with a different pool ID would produce the same value, but this is never tested

### Findings

The bug is confirmed. Every pool-share trustline exports the same `asset_id` value (`-1967220342708457407`), because the `FarmHashAsset` input never includes the `LiquidityPoolId` that distinguishes one pool-share asset from another. Pool-share assets have no code or issuer (those are properties of the underlying assets in the pool, not the pool share itself), so the only discriminating data — the pool ID — is omitted from the hash.

This means any downstream query using `asset_id` as a join or group key for trustlines will silently collapse all pool-share trustlines into a single asset bucket, producing incorrect aggregations. The `liquidity_pool_id` column is available as a workaround, but `asset_id` is the canonical surrogate key across all asset-bearing tables, and its corruption for pool shares is a structural data correctness issue.

### PoC Guidance

- **Test file**: `internal/transform/trustline_test.go`
- **Setup**: Create two `ingest.Change` entries with pool-share trustline assets referencing different `LiquidityPoolId` values (e.g., `xdr.PoolId{1,2,3,...}` and `xdr.PoolId{9,8,7,...}`)
- **Steps**: Call `TransformTrustline()` on both changes
- **Assertion**: Assert that the two resulting `TrustlineOutput.AssetID` values are NOT equal. Currently they will be equal, proving the collision. Additionally, assert that both `LiquidityPoolID` fields are different (to confirm the inputs are genuinely distinct pool shares).
