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
**Target Test File**: internal/transform/data_integrity_poc_test.go (temporary; reverted after test run)
**Test Name**: "TestBuySideNormalizedOfferPriceNotInverted"
**Test Language**: Go

### Demonstration

The test demonstrates that `extractDimOffer()` copies the raw offer-side price unchanged into `DimOffer.Price` for buy-side rows (`action="b"`), producing a value in base/counter units (ETH/native ≈ 0.514) instead of the counter/base units (native/ETH ≈ 1.945) required by the canonical `DimMarket` orientation.

This is a **distinct residual defect** from finding 021 (BaseAmount/CounterAmount swap). Even if a fix corrects only the amount fields per finding 021, the Price field would remain at `0.514` (base/counter) instead of the canonical `1.945` (counter/base). After such a partial fix, the invariant `CounterAmount == BaseAmount × Price` would break: `135.16 × 0.514 ≈ 69.5 ≠ 262.84`. Only with Price also inverted does the invariant hold: `135.16 × 1.945 ≈ 262.84`. The Price inversion is therefore an independently necessary correction.

### Relationship to Finding 021

Finding 021 demonstrated that `BaseAmount` and `CounterAmount` are swapped for buy-side offers. This finding (004) demonstrates the same root cause manifesting in the `Price` field — the raw `offer.Price` (buying/selling = base/counter) is not inverted to canonical counter/base for buy-side rows. While all three fields share the same root cause (no branching on `action` in `extractDimOffer()` lines 159–168), the Price inversion is a separately observable and independently impactful defect:

1. **Independent fix path**: A fix that only swaps BaseAmount/CounterAmount (per 021) without also inverting Price leaves the row in a state where `CounterAmount ≠ BaseAmount × Price`.
2. **Distinct downstream impact**: Analytics consuming `dim_offers.price` to compute order value or spread will get the reciprocal of the correct exchange rate for all buy-side offers, regardless of whether BaseAmount/CounterAmount are corrected.
3. **Separate observable output**: The Price field value (`0.514` vs `1.945`) is a different column with a different incorrect value than the BaseAmount/CounterAmount swap.

### Test Body

```go
package transform

import (
	"math"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestBuySideNormalizedOfferPriceNotInverted(t *testing.T) {
	// This test demonstrates that extractDimOffer() does not invert the raw
	// offer-side price for buy-side rows (action="b"), producing a Price in
	// base/counter units instead of the counter/base units required by the
	// canonical DimMarket orientation.
	//
	// This is a distinct residual defect from finding 021 (BaseAmount/CounterAmount
	// swap). Even if a fix swaps BaseAmount and CounterAmount correctly, the Price
	// field would still carry offer.Price (base/counter = ETH/native) rather than
	// the canonical counter/base (native/ETH). After such a partial fix, the
	// invariant CounterAmount == BaseAmount * Price would break.
	//
	// Fixture: selling=native, buying=ETH
	//   offer.Amount = 2628450327 stroops = 262.8450327 native (selling-asset qty)
	//   offer.Price  = 920936891/1790879058 ≈ 0.5142373444404865 (buying/selling = ETH/native)
	//
	// Canonical market (alphabetical sort): base=ETH, counter=native, action="b"
	//
	// For the canonical ETH/native market, dim_offers.price should be
	// counter/base = native/ETH = 1/offer.Price ≈ 1.944627341462424.
	// The code returns offer.Price = 0.5142373444404865 (ETH/native) unchanged.

	// 1. Construct input using the existing test helper.
	input, err := makeOfferNormalizedTestInput()
	assert.NoError(t, err)

	// 2. Run through the production code path.
	result, err := TransformOfferNormalized(input, 100)
	assert.NoError(t, err)

	// Verify canonical market orientation and action.
	assert.Equal(t, "ETH", result.Market.BaseCode, "base asset should be ETH")
	assert.Equal(t, "native", result.Market.CounterCode, "counter asset should be native")
	assert.Equal(t, "b", result.Offer.Action, "action should be 'b' (seller sells counter)")

	// 3. Assert the canonical expected Price directly.
	//
	// For a canonical ETH/native market where the seller sells native (counter):
	//   offer.Price = buying/selling = ETH/native (base/counter)
	//   Canonical Price = counter/base = native/ETH = 1/offer.Price
	//
	// This is the same convention used by the sell-side case (action="s"),
	// where offer.Price is already counter/base because the seller sells base.
	//
	// The correct canonical Price for this fixture:
	//   1 / 0.5142373444404865 ≈ 1.944627341462424

	const tolerance = 1e-6
	canonicalPrice := 1.944627341462424 // native/ETH = counter/base

	// The actual output carries the raw (un-inverted) offer.Price.
	// This assertion directly shows the Price is wrong relative to the
	// canonical market orientation.
	priceDelta := math.Abs(result.Offer.Price - canonicalPrice)
	assert.Greater(t, priceDelta, tolerance,
		"Price should differ from canonical counter/base value %.10f; got %.10f",
		canonicalPrice, result.Offer.Price)

	// Confirm the actual value equals the raw offer-side price (base/counter).
	rawOfferPrice := 0.5142373444404865 // ETH/native = base/counter
	assert.InDelta(t, rawOfferPrice, result.Offer.Price, tolerance,
		"Price carries the raw offer-side value (base/counter) instead of canonical counter/base")

	// 4. Demonstrate this is a DISTINCT defect from finding 021.
	//
	// Finding 021 showed BaseAmount and CounterAmount are swapped. Even if
	// those are corrected, Price remains wrong. After a hypothetical fix of
	// only the amount fields:
	//   BaseAmount  ≈ 135.16 (ETH) — correct
	//   CounterAmount ≈ 262.84 (native) — correct
	//   Price = 0.514... (ETH/native = base/counter) — STILL WRONG
	//
	// The invariant CounterAmount == BaseAmount * Price would yield:
	//   135.16 * 0.514... ≈ 69.51 ≠ 262.84
	//
	// Only with Price also inverted (≈ 1.9446) does the invariant hold:
	//   135.16 * 1.9446 ≈ 262.84 ✓

	correctBaseAmount := 135.16473161502083   // ETH = offer.Amount * offer.Price
	correctCounterAmount := 262.8450327       // native = offer.Amount

	// With only amount-swap fix applied and Price still raw:
	invariantWithRawPrice := correctBaseAmount * result.Offer.Price
	assert.Greater(t, math.Abs(invariantWithRawPrice-correctCounterAmount), 100.0,
		"After fixing only amounts, raw Price breaks CounterAmount == BaseAmount * Price: "+
			"%.4f * %.10f = %.4f ≠ %.4f",
		correctBaseAmount, result.Offer.Price, invariantWithRawPrice, correctCounterAmount)

	// With Price properly inverted, the invariant holds.
	invariantWithCanonicalPrice := correctBaseAmount * canonicalPrice
	assert.InDelta(t, correctCounterAmount, invariantWithCanonicalPrice, 0.01,
		"With canonical Price, invariant holds: %.4f * %.10f ≈ %.4f",
		correctBaseAmount, canonicalPrice, invariantWithCanonicalPrice)
}
```

### Test Output

```
=== RUN   TestBuySideNormalizedOfferPriceNotInverted
--- PASS: TestBuySideNormalizedOfferPriceNotInverted (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.830s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-11
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Exercises claimed bug**: NO — the PoC shows that `extractDimOffer()` copies raw `offer.Price`, but it does not prove that `DimOffer.Price` is wrong on the row the exporter actually emits today. Its "expected" value is derived from the already-corrected buy-side amounts from finding 021, not from a broken invariant in the current output.
2. **Realistic preconditions**: YES — the checked-in fixture does reach the live `action = "b"` path.
3. **Bug vs by-design**: NOT SHOWN — the repository does not document an independent contract that `dim_offers.price` must be inverted separately from the row's present base/counter orientation.
4. **Impact/severity**: N/A — rejected.
5. **In scope**: YES — if real, this would be in scope.
6. **Test correctness**: INCORRECT — the test passes by asserting that current output differs from a hypothesized canonical reciprocal and then confirming the output equals the raw offer-side price. That is a tautological check of current behavior, not a proof that a production invariant fails.
7. **Alternative explanations**: YES — the observation is fully explained by the already-confirmed buy-side row-orientation bug. On the emitted row, `price == counter_amount / base_amount` exactly: `135.16473161502083 / 262.8450327 = 0.5142373444404865`. So `price` is internally consistent with the emitted `DimOffer` amounts; the mismatch only appears when those amounts are hypothetically corrected to the canonical orientation from finding 021.
8. **Novelty**: NO — this is not an independent residual defect. It is the same branchless buy-side normalization bug already captured by `data-transform/021-normalized-offer-buy-side-base-counter-amounts-swapped.md`, just described through another field from the same row.

### Rejection Reason

The PoC does not establish a distinct `price` corruption bug. `DimOffer.Price` is internally consistent with the current emitted `BaseAmount` and `CounterAmount`; it only looks inverted against `DimMarket` because the entire buy-side row is already reversed by finding 021. The test therefore passes for the wrong reason and does not demonstrate an independent finding.

### Failed Checks

- 1
- 6
- 7
- 8
