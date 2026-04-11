# H018: Other asset exporters repeat the trustline raw-enum `asset_id` hashing bug

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Every exported `asset_id` should hash the same canonical asset tuple that the
row exposes in its JSON columns: `asset_code`, `asset_issuer`, and the exported
`asset_type` string such as `credit_alphanum4` or `native`. No entity should
feed raw XDR enum labels like `AssetTypeAssetTypeCreditAlphanum4` into
`FarmHashAsset()`.

## Mechanism

After the published trustline bug, the risk was that other callers might make
the same mistake by hashing `asset.Type.String()` instead of the normalized
string returned by `asset.Extract()`. That would silently create inconsistent
`asset_id` values across entity types for the same on-chain asset.

## Trigger

1. Export assets, trades, liquidity pools, or operation details containing a
   non-native credit asset.
2. Recompute `asset_id` from the exported `asset_code`, `asset_issuer`, and
   exported `asset_type`.
3. Compare it to the row's emitted `asset_id`.

## Target Code

- `internal/transform/asset.go:55-69` - hashes `outputAssetType` returned by
  `asset.Extract()`
- `internal/transform/trade.go:45-63` - hashes `outputSellingAssetType` /
  `outputBuyingAssetType` returned by `Extract()`
- `internal/transform/liquidity_pool.go:39-51` - hashes `assetAType` /
  `assetBType` returned by `Extract()`
- `internal/transform/operation.go:367-385` - operation detail helper hashes
  the extracted canonical `assetType`

## Evidence

This looked plausible because `trustline.go` previously used `asset.Type.String()`
and produced a confirmed cross-entity mismatch. The codebase has several other
`FarmHashAsset(...)` callers, so a repeated copy/paste mistake was a realistic
audit target.

## Anti-Evidence

All of the callers examined first normalize the asset through `asset.Extract()`
and then pass that exact exported type string into `FarmHashAsset()`. I did not
find another live caller that hashes the raw XDR enum name the way the trustline
path did.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The trustline bug does not generalize. The other live `FarmHashAsset(...)`
callers already hash canonical extracted asset-type strings, so they stay
consistent with their own exported `asset_type` columns.

### Lesson Learned

When one entity has a hashing mismatch, do not assume its siblings share it.
Audit each `FarmHashAsset(...)` caller individually: the trustline path was the
outlier because it passed `asset.Type.String()`, while the neighboring callers
already use normalized strings from `asset.Extract()`.
