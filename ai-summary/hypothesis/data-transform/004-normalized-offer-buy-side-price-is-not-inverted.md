# H004: Buy-side normalized offers keep `price` in the wrong units

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`dim_offers.price` should use the same canonical base/counter orientation as `dim_markets.base_*`, `dim_markets.counter_*`, `dim_offers.base_amount`, and `dim_offers.counter_amount`. For the checked-in fixture where the market is `ETH/native` and the offer is `action = "b"`, the normalized row should export:

- `base_amount = 135.16473161502083` ETH
- `counter_amount = 262.8450327` native
- `price = 1.944627341462424` native per ETH

so that `counter_amount == base_amount * price`.

## Mechanism

`extractDimOffer()` correctly canonicalizes the market and derives `action = "b"` when the seller is offering the counter asset, but it still copies the raw offer-side `offer.Price` directly into `dim_offers.price`. On Stellar `ManageSellOffer`, that raw price is denominated as buying-per-selling; for buy-side normalized rows the seller is selling the canonical counter asset, so the copied value is base-per-counter while the normalized schema is otherwise keyed to counter-per-base. The exported row therefore carries a plausible numeric price in the opposite unit system from the rest of the normalized market row.

## Trigger

Process any normalized offer where `sellingAsset != assets[0]` in `extractDimOffer()`. The existing fixture in `offer_normalized_test.go` already reproduces it:

1. `Selling = native`
2. `Buying = ETH`
3. `Amount = 2628450327` stroops (`262.8450327` native)
4. `Price = 920936891/1790879058 = 0.5142373444404865`

The current export emits `price = 0.5142373444404865`, but the canonical `ETH/native` row should emit the reciprocal `1.944627341462424`.

## Target Code

- `internal/transform/offer_normalized.go:139-167` — hashes the offer, derives `action`, and assigns `base_amount`, `counter_amount`, and `price`
- `internal/transform/schema.go:321-329` — `DimOffer` names the exported fields as canonical `base_amount`, `counter_amount`, and `price`
- `internal/transform/offer_normalized_test.go:97-114` — checked-in fixture locks in `base_code = "ETH"`, `counter_code = "native"`, `action = "b"`, and `price = 0.5142373444404865`

## Evidence

The current fixture is internally inconsistent even before any PoC: `Market.BaseCode` is `"ETH"` and `Market.CounterCode` is `"native"`, yet `Offer.Price` remains `0.5142373444404865`, which is the raw offer-side price copied from `TransformOffer()` rather than the canonical `counter/base` ratio implied by the normalized market. Using the same fixture values, `counter_amount / base_amount = 262.8450327 / 135.16473161502083 = 1.944627341462424`, not `0.5142373444404865`.

## Anti-Evidence

The strongest alternative explanation is that downstream consumers may intentionally interpret `dim_offers.price` as the raw Stellar offer-side price regardless of canonical market orientation. But if that is the intended contract, `price` is the lone field in `dim_offers` that does not follow the canonical `base_*` / `counter_*` naming used everywhere else in the normalized row.
