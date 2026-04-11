# H002: Buy-side normalized offers swap `base_amount` and `counter_amount`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`dim_offers.base_amount` should always describe the amount of the canonical base asset from `dim_markets`, and `counter_amount` should always describe the amount of the canonical counter asset. When `extractDimOffer()` marks an offer as `action = "b"` because the seller is offering the counter asset to buy the base asset, the exported amounts should flip accordingly: `base_amount` should be the bought quantity and `counter_amount` should be the sold quantity.

For the checked-in fixture in `offer_normalized_test.go`, the market is `ETH/native`, the offer sells `262.8450327` native, and the price implies buying `135.16473161502083` ETH. The correct normalized row should therefore be:

- `base_amount = 135.16473161502083`
- `counter_amount = 262.8450327`

## Mechanism

`extractDimOffer()` correctly derives `action = "b"` when `sellingAsset != assets[0]`, meaning the offer sells the counter asset rather than the base asset. But it still unconditionally writes `BaseAmount: offer.Amount` and `CounterAmount: float64(offer.Amount) * offer.Price`, so every buy-side row labels the sold counter quantity as `base_amount` and the bought base quantity as `counter_amount`.

## Trigger

Process any offer where the selling asset sorts after the buying asset in the canonical market ordering, for example the fixture in `internal/transform/offer_normalized_test.go`:

1. `Selling = native`
2. `Buying = ETH`
3. `Amount = 2628450327`
4. `Price = 920936891 / 1790879058`

The exported row becomes `action:"b", base_amount:262.8450327, counter_amount:135.16473161502083` even though the base asset is ETH and the counter asset is native.

## Target Code

- `internal/transform/offer_normalized.go:139-168` — computes `action` from sorted assets but does not use it when assigning `BaseAmount` / `CounterAmount`
- `internal/transform/schema.go:321-329` — `DimOffer` names the fields as canonical market-side amounts, not sold/bought amounts
- `internal/transform/offer_normalized_test.go:98-114` — current expected output already locks in the swapped values for a buy-side offer

## Evidence

The code already distinguishes buy- vs sell-side offers with the explicit `Action` field, so the function knows when `offer.Amount` is denominated in the counter asset rather than the base asset. The checked-in test fixture makes the mismatch concrete: `Market.BaseCode` is `ETH`, `Action` is `"b"`, but `BaseAmount` is still the sold native quantity.

## Anti-Evidence

The current unit test asserts the existing output, so this is not a regression that slipped past tests; it is a likely longstanding contract bug in the normalized-offer export. If downstream consumers intentionally reinterpret `base_amount` / `counter_amount` as sold/bought amounts instead of canonical market-side amounts, the field names themselves are misleading and reviewer confirmation of intended semantics will still be needed.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `TransformOfferNormalized` → `extractAssets` → `extractDimMarket` → `extractDimOffer`. The market construction in `extractDimMarket` correctly sorts assets alphabetically to establish a canonical base/counter pair (`assets[0]` = base, `assets[1]` = counter). The `extractDimOffer` function correctly re-sorts the same assets and derives `action = "b"` when the selling asset is the counter asset (not `assets[0]`). However, the amount assignment on lines 165–166 unconditionally sets `BaseAmount: offer.Amount` and `CounterAmount: float64(offer.Amount) * offer.Price`, ignoring the `action` flag entirely. For buy-side offers, `offer.Amount` is the quantity of the counter asset (the selling asset), not the base asset.

### Code Paths Examined

- `internal/transform/offer_normalized.go:16-57` — `TransformOfferNormalized` calls `extractAssets`, `extractDimMarket`, and `extractDimOffer` in sequence
- `internal/transform/offer_normalized.go:60-98` — `extractAssets` returns `(buyingAsset, sellingAsset)` with buying first; `offer.Amount` in the XDR always denominates the selling asset
- `internal/transform/offer_normalized.go:101-136` — `extractDimMarket` sorts assets alphabetically; `assets[0]` → base, `assets[1]` → counter; confirms `BaseCode = "ETH"`, `CounterCode = "native"` for the test fixture
- `internal/transform/offer_normalized.go:139-168` — `extractDimOffer` computes `action` correctly but does not branch on it for amount assignment; lines 165–166 always assign `BaseAmount = offer.Amount` (selling asset quantity) and `CounterAmount = offer.Amount * price` (buying asset quantity)
- `internal/transform/offer.go:44-90` — `TransformOffer` sets `Amount = ConvertStroopValueToReal(offerEntry.Amount)` (the selling asset amount in real units) and `Price = PriceN/PriceD` (buying per selling)
- `internal/transform/schema.go:320-330` — `DimOffer` struct fields `BaseAmount` and `CounterAmount` clearly correspond to the market's canonical base and counter assets
- `internal/transform/offer_normalized_test.go:92-124` — Test fixture: selling native, buying ETH, `Action: "b"`, `BaseCode: "ETH"`, but `BaseAmount: 262.84` (native quantity) — confirms the swap is locked in by the test

### Findings

The bug is confirmed: for every buy-side offer (`action = "b"`), the `BaseAmount` field contains the counter-asset quantity and the `CounterAmount` field contains the base-asset quantity. This is a direct financial data corruption because:

1. **`offer.Amount`** (XDR) is always the selling asset quantity. For sell-side offers the selling asset IS the base asset, so `BaseAmount = offer.Amount` is correct. For buy-side offers the selling asset is the counter asset, so `BaseAmount = offer.Amount` is wrong.

2. **`offer.Amount * offer.Price`** computes the buying asset quantity (since `Price = buying/selling`). For sell-side, buying = counter, so `CounterAmount = Amount * Price` is correct. For buy-side, buying = base, so this should be `BaseAmount`, not `CounterAmount`.

3. The unit test at line 111 confirms the swapped state: `BaseAmount: 262.8450327` (native quantity) while `Market.BaseCode` is `"ETH"` — these are semantically inconsistent.

4. Approximately half of all normalized offers in any production export (all buy-side offers) will have their base and counter amounts transposed. Any downstream analytics computing market depth, volume by asset, or VWAP from these columns would silently consume wrong values.

### PoC Guidance

- **Test file**: `internal/transform/offer_normalized_test.go`
- **Setup**: Use the existing `makeOfferNormalizedTestInput()` fixture (selling native, buying ETH) which produces `action = "b"`
- **Steps**: Call `TransformOfferNormalized(hardCodedInput, 100)` and inspect the `Offer` field
- **Assertion**: Assert that `result.Offer.BaseAmount` equals the ETH (base asset) quantity (`≈135.1647`) and `result.Offer.CounterAmount` equals the native (counter asset) quantity (`262.8450327`). The current test asserts the opposite, confirming the bug. The PoC should demonstrate the discrepancy by showing `result.Market.BaseCode == "ETH"` but `result.Offer.BaseAmount == 262.8450327` (which is the native/counter quantity, not the ETH/base quantity).
