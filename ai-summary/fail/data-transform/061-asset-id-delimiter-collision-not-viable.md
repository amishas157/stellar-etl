# H061: Delimiter-free `asset_id` hashing aliases distinct assets

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Different `(asset_code, asset_issuer, asset_type)` tuples should always export
different `asset_id` values. A helper that hashes asset identity should not
allow two valid Stellar assets to collide before the hash even runs.

## Mechanism

`FarmHashAsset()` hashes the delimiter-free preimage
`assetCode + assetIssuer + assetType`, which initially looked vulnerable to
boundary-shift aliases between the code, issuer, and type segments. If two
different assets could produce the same concatenated string, every table that
relies on `asset_id` would silently conflate them.

## Trigger

Construct two valid assets whose code/issuer/type boundaries shift while the raw
concatenated string stays identical, then export any asset-bearing table such as
trustlines, offers, or pools.

## Target Code

- `internal/transform/asset.go:72-76` — `FarmHashAsset()` builds the preimage without separators
- `internal/transform/asset.go:55-69` — `transformSingleAsset()` feeds the helper from extracted asset fields
- `internal/transform/liquidity_pool.go:39-51` — pool reserves reuse the same `asset_id` helper
- `internal/transform/trade.go:45-63` — trade asset IDs reuse the same helper

## Evidence

The helper really does hash a plain concatenation with no delimiter or length
prefix, so at first glance it appears susceptible to ambiguous parsing.

## Anti-Evidence

The domain constraints block the suspected alias. Issuers are fixed-length
account IDs, and the asset type suffix is one of a small fixed set of strings,
so same-type aliases require equal code lengths and therefore identical segment
boundaries. Cross-type aliases would need valid credit-asset codes to absorb the
different type suffixes, which does not line up with the actual asset encoding
arms.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The delimiter-free preimage looks suspicious in isolation, but the Stellar
asset-domain constraints remove the ambiguity that would be needed to create a
live alias. The candidate depends on segment-boundary freedom that valid asset
encodings do not actually provide.

### Lesson Learned

When evaluating a delimiter-free hash preimage, check whether the component
lengths are already fixed by the protocol. A scary-looking concatenation bug is
only live if the domain really allows multiple valid segmentations.
