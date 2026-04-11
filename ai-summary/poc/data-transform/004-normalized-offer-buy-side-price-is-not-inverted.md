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

The test demonstrates that `extractDimOffer()` copies the raw offer-side price unchanged into `DimOffer.Price` for buy-side rows (`action="b"`), producing a value in base/counter units (ETH/native ≈ 0.514) instead of the counter/base units (native/ETH ≈ 1.945) required by the canonical `DimMarket` orientation. This is a distinct observable output defect from the BaseAmount/CounterAmount swap demonstrated in finding 021: the Price field carries the reciprocal of the expected value. The existing test fixture at `offer_normalized_test.go:105-113` locks in this buggy value, masking the defect. Any downstream analytics using `dim_offers.price` with `dim_markets.base_code`/`counter_code` will interpret the price ratio in the wrong direction for all buy-side offers.

### Relationship to Finding 021

Finding `data-transform/021-normalized-offer-buy-side-base-counter-amounts-swapped` demonstrated that `BaseAmount` and `CounterAmount` are swapped for buy-side offers. This finding (004) demonstrates the same root cause manifesting in the `Price` field: the raw `offer.Price` (buying/selling = base/counter) is not inverted to canonical counter/base for buy-side rows. While all three fields share the same root cause (no branching on `action` in `extractDimOffer()` lines 159–168), the Price inversion is a separately observable and independently impactful defect — a fix that only swaps BaseAmount/CounterAmount without also inverting Price would leave the row in a state where `CounterAmount ≠ BaseAmount × Price`.

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
	// base/counter units instead of the counter/base units implied by the
	// canonical DimMarket orientation.
	//
	// The existing finding 021 demonstrated that BaseAmount and CounterAmount
	// are swapped for buy-side offers. This test focuses specifically on the
	// Price field, which suffers from the same root cause but is a distinct
	// observable output defect: it carries the reciprocal of the expected value.
	//
	// Fixture: selling=native, buying=ETH
	//   offer.Amount = 2628450327 stroops = 262.8450327 native (selling-asset qty)
	//   offer.Price  = 920936891/1790879058 ≈ 0.5142373444404865 (buying/selling = ETH/native)
	//
	// Canonical market (alphabetical sort): base=ETH, counter=native, action="b"
	//
	// For the canonical ETH/native market, Price should be counter/base = native/ETH.
	// Since offer.Price = ETH/native (base/counter), the correct canonical Price
	// is 1.0/offer.Price ≈ 1.944627341462424 (native per ETH).
	//
	// The code returns offer.Price = 0.5142373444404865 (ETH per native) unchanged.

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

	// 3. Demonstrate the Price bug.
	//
	// For a canonical market where Price = counter/base:
	//   Correct Price = native/ETH = 1/offer.Price ≈ 1.944627341462424
	//   Actual Price  = ETH/native = offer.Price   ≈ 0.5142373444404865
	//
	// The raw offer.Price is buying/selling = ETH/native, which is base/counter.
	// The canonical DimMarket schema expects counter/base, so Price should be inverted.

	correctPrice := 1.944627341462424      // native/ETH = 1/offer.Price (counter/base)
	rawOfferPrice := 0.5142373444404865     // ETH/native = offer.Price (base/counter)
	const tolerance = 1e-6

	// Assert the output matches the raw (wrong) offer-side price, not the correct canonical price.
	assert.InDelta(t, rawOfferPrice, result.Offer.Price, tolerance,
		"Price should contain the raw offer-side value (base/counter = ETH/native)")

	// Assert the output does NOT match the correct canonical price.
	priceIsWrong := math.Abs(result.Offer.Price - correctPrice) > tolerance
	assert.True(t, priceIsWrong,
		"Price should differ from correct canonical value: got %v, correct is %v",
		result.Offer.Price, correctPrice)

	// Verify that the correct price would be the reciprocal.
	reciprocal := 1.0 / result.Offer.Price
	assert.InDelta(t, correctPrice, reciprocal, tolerance,
		"The reciprocal of the actual Price should equal the correct canonical Price")

	// The sell-side case (action="s") would be correct because offer.Price
	// is already in the right orientation when the seller sells the base asset.
	// This bug only manifests for buy-side offers where the raw price needs inversion.
}
```

### Test Output

```
=== RUN   TestBuySideNormalizedOfferPriceNotInverted
--- PASS: TestBuySideNormalizedOfferPriceNotInverted (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.817s
```

---

## Final Review — Needs Revision

**Date**: 2026-04-11
**Final review by**: gpt-5.4, high

### What Needs Fixing

The underlying code defect is real, but this hypothesis is not currently framed as a distinct finding. `extractDimOffer()` does not have a price-only bug: the same `action == "b"` branch also leaves `base_amount` and `counter_amount` in raw offer orientation. The repository's condensed success index already contains `data-transform/021-normalized-offer-buy-side-base-counter-amounts-swapped.md`, so this PoC currently overlaps an already-recorded broader buy-side canonicalization issue instead of establishing a separate new finding.

The PoC metadata is also stale: the document points to `internal/transform/data_integrity_poc_test.go`, which does not exist in-tree. I had to create a temporary test file to rerun the production path, then revert it.

### Revision Instructions

1. Reframe this as a duplicate/subsumed issue under the broader buy-side canonicalization bug, or explain why `price` remains a distinct residual defect after accounting for the already-recorded `base_amount` / `counter_amount` swap.
2. If you keep this file as a standalone candidate, update the hypothesis title, mechanism, and impact to describe the full buy-side canonicalization failure rather than only the price inversion.
3. Replace the stale `Target Test File` metadata with a real target (for example `internal/transform/offer_normalized_test.go`) or explicitly say that the PoC requires a temporary test file.
4. Keep the PoC focused on expected-vs-actual canonical semantics tied to `Market.BaseCode`, `Market.CounterCode`, and `Offer.Action`, not just "the current output is wrong because it differs from another calculation."

### Checks Passed So Far

- Code path exists: `extractDimOffer()` assigns identical `BaseAmount`, `CounterAmount`, and `Price` fields for both `action == "s"` and `action == "b"` in `internal/transform/offer_normalized.go:159-168`.
- Trigger is realistic: the checked-in fixture in `internal/transform/offer_normalized_test.go:65-88` creates a real buy-side normalized row (`selling=native`, `buying=ETH`, `action="b"`).
- Bug vs by-design: the output row labels the market as `base=ETH`, `counter=native`, but the buy-side offer values remain in raw selling/buying orientation, so the field semantics do not match the exported labels.
- Impact is real: the exported normalized offer row can silently misstate canonical offer quantities and price for downstream analytics.
