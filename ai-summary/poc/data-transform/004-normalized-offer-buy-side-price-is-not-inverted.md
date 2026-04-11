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

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`extractDimOffer()` uses identical field assignments for both `action = "s"` and `action = "b"` cases, but the semantics of `offer.Amount` (selling asset quantity) and `offer.Price` (buying-per-selling) are relative to the raw offer's selling asset, not the canonical market orientation. For `action = "s"` the seller sells the base asset, so the assignments happen to be correct. For `action = "b"` the seller sells the counter asset, so all three derived fields — `BaseAmount`, `CounterAmount`, and `Price` — carry values in the wrong canonical orientation. The bug is broader than the hypothesis title suggests: not only is Price inverted, but BaseAmount and CounterAmount are also swapped.

### Code Paths Examined

- `internal/transform/offer.go:44-66,79-101` — `TransformOffer` converts `OfferEntry.Amount` (selling asset stroops) via `ConvertStroopValueToReal` → `offer.Amount` (float64), and `Price.N/Price.D` → `offer.Price` (float64). These are always denominated as *selling asset amount* and *buying-per-selling price*.
- `internal/transform/offer_normalized.go:102-136` — `extractDimMarket` alphabetically sorts `[buyingAsset, sellingAsset]`; `assets[0]` = base, `assets[1]` = counter. For the test fixture: base = `"ETH:issuer"`, counter = `"native:"`.
- `internal/transform/offer_normalized.go:149-157` — `extractDimOffer` performs the same alphabetical sort, then sets `action = "s"` if `sellingAsset == assets[0]` (seller sells base), otherwise `action = "b"` (seller sells counter). For the test fixture: `sellingAsset = "native:" != assets[0] = "ETH:..."`, so `action = "b"`. Correct.
- `internal/transform/offer_normalized.go:159-168` — Field assignments (same for both actions):
  - `BaseAmount: offer.Amount` — for `action = "b"`, this is the counter asset (native) quantity → **WRONG**
  - `CounterAmount: float64(offer.Amount) * offer.Price` — for `action = "b"`, this is native × ETH/native = ETH (the base asset) quantity → **WRONG**
  - `Price: offer.Price` — for `action = "b"`, this is ETH/native (base/counter), but should be native/ETH (counter/base) for consistency with the "s" case → **WRONG**
- `internal/transform/offer_normalized_test.go:105-113` — Test fixture encodes the current (buggy) values: `BaseAmount: 262.8450327` (native), `CounterAmount: 135.16473161502083` (ETH), `Price: 0.5142373444404865` (ETH/native)

### Findings

**All three derived fields are semantically wrong for `action = "b"`.**

For `action = "s"` (seller sells base ETH, buys counter native):
- `offer.Amount` = ETH quantity, `offer.Price` = native/ETH
- `BaseAmount = 262.84` ETH ✓, `CounterAmount = 135.16` native ✓, `Price = 0.514` native/ETH ✓

For `action = "b"` (seller sells counter native, buys base ETH):
- `offer.Amount` = native quantity, `offer.Price` = ETH/native
- `BaseAmount = 262.84` (native quantity, but labeled as base=ETH) ✗
- `CounterAmount = 135.16` (ETH quantity, but labeled as counter=native) ✗
- `Price = 0.514` (ETH/native = base/counter, should be counter/base) ✗

Correct values for the test fixture (market ETH/native, action="b"):
- `BaseAmount = 135.16473161502083` (ETH quantity = offer.Amount × offer.Price)
- `CounterAmount = 262.8450327` (native quantity = offer.Amount)
- `Price = 1.944627341462424` (native/ETH = 1/offer.Price)

The mathematical invariant `CounterAmount = BaseAmount × Price` holds under BOTH the current (buggy) and correct assignments, but the semantic meaning is wrong. Any downstream BigQuery query joining `dim_offers.base_amount` with `dim_markets.base_code` to aggregate "total ETH volume" will silently include native amounts from buy-side rows, corrupting financial analytics.

The test fixture at `offer_normalized_test.go:105-113` locks in the buggy behavior, which is why this has not been caught — the test passes with the wrong values.

### PoC Guidance

- **Test file**: `internal/transform/offer_normalized_test.go` (append a new test function)
- **Setup**: Use the existing `makeOfferNormalizedTestInput()` which creates a selling=native, buying=ETH offer with Amount=2628450327 and Price=920936891/1790879058. This already triggers the buy-side path (`action = "b"`).
- **Steps**: Call `TransformOfferNormalized()` on the input. Extract `result.Offer.BaseAmount`, `result.Offer.CounterAmount`, and `result.Offer.Price`. Also extract `result.Market.BaseCode` and `result.Market.CounterCode` for context.
- **Assertion**: Assert that `result.Market.BaseCode` is `"ETH"` (verified). Then assert `result.Offer.BaseAmount` ≈ `135.16473161502083` (the ETH quantity) — this currently fails because the code returns `262.8450327` (the native quantity). Also assert `result.Offer.Price` ≈ `1.944627341462424` (counter/base) — currently fails, returns `0.5142373444404865` (base/counter).
- **Fix**: In `extractDimOffer()` at `offer_normalized.go:159-168`, add a branch for `action == "b"` that swaps the assignments: `BaseAmount = offer.Amount * offer.Price`, `CounterAmount = offer.Amount`, `Price = 1.0 / offer.Price`. The existing test fixture expected values also need updating.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestBuySideNormalizedOfferCanonicalOrientationBug"
**Test Language**: Go

### Demonstration

The test demonstrates the broader buy-side canonicalization bug in `extractDimOffer()`. `TransformOffer()` exposes `offer.Amount` as the selling-asset quantity and `offer.Price` as `buying / selling`. When `extractDimOffer()` determines `action = "b"` (the seller sells the canonical counter asset), it fails to remap those raw offer-side values into canonical `DimMarket` base/counter units. All three derived fields are affected: BaseAmount returns 262.8450327 (the native/counter quantity, not ETH/base), CounterAmount returns 135.16 (the ETH/base quantity, not native/counter), and Price returns 0.514 (base/counter = ETH/native, not counter/base = native/ETH). The mathematical invariant `CounterAmount = BaseAmount × Price` still holds with the buggy values, making the bug insidious — the row is internally consistent but semantically wrong relative to the canonical market labels.

### Test Body

```go
func TestBuySideNormalizedOfferCanonicalOrientationBug(t *testing.T) {
	// This test demonstrates that extractDimOffer() does not remap raw offer-side
	// values into canonical base/counter orientation for buy-side rows (action="b").
	//
	// TransformOffer() exposes:
	//   offer.Amount = selling-asset quantity (in XLM for this fixture)
	//   offer.Price  = buying-per-selling   (ETH per native for this fixture)
	//
	// extractDimOffer() alphabetically sorts assets to determine the canonical
	// market: base=ETH, counter=native. Because sellingAsset (native) != assets[0]
	// (ETH), action="b" is correctly assigned. However, the field assignments are
	// identical for both "s" and "b" branches, so the raw offer-side values are
	// never remapped:
	//   BaseAmount    = offer.Amount              (native qty, labeled as base=ETH) WRONG
	//   CounterAmount = offer.Amount * offer.Price (ETH qty, labeled as counter=native) WRONG
	//   Price         = offer.Price               (ETH/native = base/counter) WRONG
	//
	// The correct canonical values for the ETH/native market with action="b" are:
	//   BaseAmount    = offer.Amount * offer.Price (ETH qty ≈ 135.16)
	//   CounterAmount = offer.Amount               (native qty = 262.84)
	//   Price         = 1.0 / offer.Price          (native/ETH ≈ 1.9446)

	// 1. Construct input using the existing test helper.
	//    Selling = native (counter after alphabetical sort)
	//    Buying  = ETH    (base after alphabetical sort)
	//    Amount  = 2628450327 stroops = 262.8450327 native (the selling-asset qty)
	//    Price   = 920936891/1790879058 ≈ 0.5142373444404865 (buying/selling = ETH/native)
	input, err := makeOfferNormalizedTestInput()
	assert.NoError(t, err)

	// 2. Run through the production code path.
	result, err := TransformOfferNormalized(input, 100)
	assert.NoError(t, err)

	// Verify the canonical market orientation.
	assert.Equal(t, "ETH", result.Market.BaseCode, "base asset should be ETH")
	assert.Equal(t, "native", result.Market.CounterCode, "counter asset should be native")
	assert.Equal(t, "b", result.Offer.Action, "action should be 'b' (seller sells counter)")

	// 3. Demonstrate the bug: ALL THREE derived fields are in the wrong orientation.
	//
	// Correct canonical values:
	correctBaseAmount := 135.16473161502083 // ETH quantity = offer.Amount × offer.Price
	correctCounterAmount := 262.8450327     // native quantity = offer.Amount
	correctPrice := 1.944627341462424       // native/ETH = 1/offer.Price (counter/base)

	const tolerance = 1e-6

	// BUG 1: BaseAmount returns 262.84 (native/counter qty) instead of 135.16 (ETH/base qty).
	// The code blindly copies offer.Amount (the selling-asset qty) without checking action.
	baseAmountIsWrong := math.Abs(result.Offer.BaseAmount-correctBaseAmount) > tolerance
	assert.True(t, baseAmountIsWrong,
		"BaseAmount should be wrong: got %v, correct canonical value is %v",
		result.Offer.BaseAmount, correctBaseAmount)
	assert.InDelta(t, 262.8450327, result.Offer.BaseAmount, tolerance,
		"BaseAmount contains the native (counter) quantity instead of ETH (base)")

	// BUG 2: CounterAmount returns 135.16 (ETH/base qty) instead of 262.84 (native/counter qty).
	// The code computes offer.Amount * offer.Price, which for this fixture yields the buying-asset qty.
	counterAmountIsWrong := math.Abs(result.Offer.CounterAmount-correctCounterAmount) > tolerance
	assert.True(t, counterAmountIsWrong,
		"CounterAmount should be wrong: got %v, correct canonical value is %v",
		result.Offer.CounterAmount, correctCounterAmount)
	assert.InDelta(t, 135.16473161502083, result.Offer.CounterAmount, tolerance,
		"CounterAmount contains the ETH (base) quantity instead of native (counter)")

	// BUG 3: Price returns 0.514 (ETH/native = base/counter) instead of 1.945 (native/ETH = counter/base).
	// The code copies offer.Price (buying-per-selling) without inverting for the "b" case.
	priceIsWrong := math.Abs(result.Offer.Price-correctPrice) > tolerance
	assert.True(t, priceIsWrong,
		"Price should be wrong: got %v, correct canonical value is %v",
		result.Offer.Price, correctPrice)
	assert.InDelta(t, 0.5142373444404865, result.Offer.Price, tolerance,
		"Price contains the raw offer-side value (base/counter) instead of counter/base")

	// The mathematical invariant CounterAmount ≈ BaseAmount × Price holds even with
	// the buggy values — this is why the bug is insidious. The row is internally
	// consistent but semantically wrong relative to the canonical market labels.
	invariant := result.Offer.BaseAmount * result.Offer.Price
	assert.InDelta(t, result.Offer.CounterAmount, invariant, tolerance,
		"CounterAmount should equal BaseAmount*Price even with buggy values")

	// Any BigQuery query joining dim_offers.base_amount with dim_markets.base_code
	// to aggregate "total ETH volume" will silently include native amounts from
	// buy-side rows, corrupting financial analytics.
}
```

### Test Output

```
=== RUN   TestBuySideNormalizedOfferCanonicalOrientationBug
--- PASS: TestBuySideNormalizedOfferCanonicalOrientationBug (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.694s
```
