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
**Test Name**: "TestBuySideNormalizedOfferFieldsAreSwapped"
**Test Language**: Go

### Demonstration

The test calls `TransformOfferNormalized()` with the existing test fixture (selling=native, buying=ETH, action="b") and verifies that the output values do NOT match the correct canonical orientation. BaseAmount returns 262.8450327 (the native/counter quantity) instead of the correct ETH/base quantity of 135.16. CounterAmount returns 135.16 (ETH/base) instead of the correct native/counter quantity of 262.84. Price returns 0.514 (base/counter = ETH/native) instead of the correct 1.945 (counter/base = native/ETH). All three fields are semantically wrong for buy-side offers while the mathematical invariant `CounterAmount = BaseAmount × Price` still holds, making the bug insidious.

### Test Body

```go
func TestBuySideNormalizedOfferFieldsAreSwapped(t *testing.T) {
	// 1. Construct input using the existing test helper.
	//    Selling = native (counter after sort), Buying = ETH (base after sort)
	//    Amount = 2628450327 stroops = 262.8450327 native
	//    Price  = 920936891/1790879058 ≈ 0.5142373444404865 (ETH per native)
	input, err := makeOfferNormalizedTestInput()
	assert.NoError(t, err)

	// 2. Run through the production code path.
	result, err := TransformOfferNormalized(input, 100)
	assert.NoError(t, err)

	// Verify the market orientation: base=ETH, counter=native
	assert.Equal(t, "ETH", result.Market.BaseCode, "base asset should be ETH")
	assert.Equal(t, "native", result.Market.CounterCode, "counter asset should be native")
	assert.Equal(t, "b", result.Offer.Action, "action should be 'b' (seller sells counter)")

	// 3. Demonstrate the bug: the output values do NOT match canonical orientation.
	//
	//    For a canonical ETH/native market with action="b":
	//      - The seller sells native (counter), so offer.Amount is the native quantity.
	//      - BaseAmount SHOULD be the ETH quantity ≈ 135.16473161502083
	//      - CounterAmount SHOULD be the native quantity = 262.8450327
	//      - Price SHOULD be counter/base = native/ETH ≈ 1.944627341462424
	//
	//    But the code produces:
	//      - BaseAmount    = 262.8450327        (native quantity — WRONG, should be ETH)
	//      - CounterAmount = 135.16473161502083  (ETH quantity — WRONG, should be native)
	//      - Price         = 0.5142373444404865  (ETH/native — WRONG, should be native/ETH)

	correctBaseAmount := 135.16473161502083 // ETH quantity
	correctCounterAmount := 262.8450327     // native quantity
	correctPrice := 1.944627341462424       // native per ETH (counter/base)

	const tolerance = 1e-6

	// BUG 1: BaseAmount returns the native quantity (262.84) instead of ETH (135.16).
	// Assert the output does NOT match the correct canonical value.
	baseAmountIsWrong := math.Abs(result.Offer.BaseAmount-correctBaseAmount) > tolerance
	assert.True(t, baseAmountIsWrong,
		"BaseAmount should be wrong: got %v, which unexpectedly matches correct value %v",
		result.Offer.BaseAmount, correctBaseAmount)
	// Assert the output instead matches the counter-asset quantity (native amount).
	assert.InDelta(t, 262.8450327, result.Offer.BaseAmount, tolerance,
		"BaseAmount incorrectly contains the native (counter) quantity instead of ETH (base)")

	// BUG 2: CounterAmount returns the ETH quantity (135.16) instead of native (262.84).
	counterAmountIsWrong := math.Abs(result.Offer.CounterAmount-correctCounterAmount) > tolerance
	assert.True(t, counterAmountIsWrong,
		"CounterAmount should be wrong: got %v, which unexpectedly matches correct value %v",
		result.Offer.CounterAmount, correctCounterAmount)
	assert.InDelta(t, 135.16473161502083, result.Offer.CounterAmount, tolerance,
		"CounterAmount incorrectly contains the ETH (base) quantity instead of native (counter)")

	// BUG 3: Price returns 0.514 (base/counter = ETH/native) instead of 1.945 (counter/base = native/ETH).
	priceIsWrong := math.Abs(result.Offer.Price-correctPrice) > tolerance
	assert.True(t, priceIsWrong,
		"Price should be wrong: got %v, which unexpectedly matches correct value %v",
		result.Offer.Price, correctPrice)
	assert.InDelta(t, 0.5142373444404865, result.Offer.Price, tolerance,
		"Price incorrectly contains the raw offer-side price (base/counter) instead of counter/base")

	// Verify the mathematical invariant CounterAmount ≈ BaseAmount × Price still holds
	// even with the swapped values — this is why the bug is insidious.
	invariant := result.Offer.BaseAmount * result.Offer.Price
	assert.InDelta(t, result.Offer.CounterAmount, invariant, tolerance,
		"CounterAmount should equal BaseAmount*Price even with buggy values")
}
```

### Test Output

```
=== RUN   TestBuySideNormalizedOfferFieldsAreSwapped
--- PASS: TestBuySideNormalizedOfferFieldsAreSwapped (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.626s
```
