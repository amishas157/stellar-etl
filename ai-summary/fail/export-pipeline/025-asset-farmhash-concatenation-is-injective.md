# H025: Delimiter-free `FarmHashAsset` concatenation does not create valid asset-ID aliasing

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Distinct Stellar assets should remain distinguishable when exported through `asset_id`, `selling_asset_id`, `buying_asset_id`, and the other `FarmHashAsset()` call sites. A valid asset tuple `(asset_code, asset_issuer, asset_type)` should never alias another valid tuple before hashing.

## Mechanism

`FarmHashAsset()` builds its hash input with `fmt.Sprintf("%s%s%s", assetCode, assetIssuer, assetType)` and no delimiter, which initially suggests that different tuples could concatenate to the same byte string and therefore export the same `asset_id`. If that were reachable for legitimate Stellar assets, downstream joins across trades, offers, trustlines, and asset tables could silently merge unrelated assets.

## Trigger

Construct two different asset tuples whose raw string concatenations would need to match across `assetCode + assetIssuer + assetType`, then export any entity that calls `FarmHashAsset()`.

## Target Code

- `internal/transform/asset.go:72-76` — `FarmHashAsset()` concatenates asset code, issuer, and type without separators before hashing.
- `internal/transform/trade.go:45-63` — trade export derives selling/buying asset IDs from the same helper.
- `internal/transform/offer.go` — offer export reuses the same helper for asset IDs.

## Evidence

The helper is plainly delimiter-free, so a generic "string concatenation collision" concern is reasonable on first read. The same helper feeds multiple exported tables, so any alias would be cross-surface corruption rather than an isolated column bug.

## Anti-Evidence

For legitimate Stellar assets, the issuer is a fixed-length StrKey account string for credit assets, while native assets always use empty code/issuer plus the distinct `native` type marker. The canonical asset-type strings (`native`, `credit_alphanum4`, `credit_alphanum12`) also prevent cross-type re-partitioning into another valid tuple.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Although the helper omits delimiters, the valid Stellar asset domain makes the concatenation effectively injective: fixed-length issuer encoding and distinct type strings prevent two legitimate asset tuples from producing the same preimage.

### Lesson Learned

Delimiter-free concatenation is only a live export bug when the component domains admit ambiguous decompositions. For Stellar assets, protocol-level length and type constraints matter more than the generic string-pattern smell.
