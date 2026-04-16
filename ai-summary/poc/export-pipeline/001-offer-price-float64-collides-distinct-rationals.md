# H001: Offer-state `price` collapses distinct on-chain prices

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`offers.price` should remain distinct for distinct on-chain `xdr.Price` values, so downstream consumers that read the scalar price column can distinguish adjacent offer states without having to reconstruct the rational from companion fields.

## Mechanism

`TransformOffer()` exports the scalar `Price` as `float64(outputPriceN) / float64(outputPriceD)`. Near `xdr.Int32` bounds, valid neighboring rationals differ by less than one `float64` ulp, so distinct prices collapse to the same IEEE-754 value even though `price_n` and `price_d` remain different. For example, `2147478646/2147478647` and `2147478647/2147478648` both export as `0.9999999995343376`.

## Trigger

Export two offer ledger entries whose `OfferEntry.Price` values are `2147478646/2147478647` and `2147478647/2147478648`. The resulting `offers` rows preserve different `price_n`/`price_d`, but the scalar `price` field is identical in both rows.

## Target Code

- `internal/transform/offer.go:49-66` — validates `Price.N` / `Price.D` and computes the scalar `outputPrice`
- `internal/transform/offer.go:79-101` — emits `Price`, `PriceN`, and `PriceD` into `OfferOutput`
- `cmd/export_ledger_entry_changes.go:204-212` — exports `TransformOffer()` rows directly into the live `offers` dataset

## Evidence

The live transform uses raw `float64` division for the scalar column instead of an exact decimal or string path. A local calculation against the production conversion showed a concrete collision: `(2147478646,2147478647)` and `(2147478647,2147478648)` both round to the same exported double even though the on-chain prices are distinct.

## Anti-Evidence

The exporter also preserves exact `price_n` and `price_d`, so consumers that ignore `price` can reconstruct the true rational. But `price` is still emitted as a first-class JSON/Parquet field with no warning that different on-chain prices can collapse onto the same numeric value.

---

## Review

**Verdict**: VIABLE
**Severity**: Low
**Date**: 2026-04-16
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (fail/046 covers DimOfferID `%f` formatting in dormant normalized-offer code, a different mechanism and code path)

### Trace Summary

Traced from `export_ledger_entry_changes.go:178` through `TransformOffer()` in `offer.go:63-66` where `float64(outputPriceN) / float64(outputPriceD)` computes the scalar price. Both `outputPriceN` and `outputPriceD` are `int32` (valid range [1, 2147483647]). For rationals near 1.0 with both components near int32 max, the difference between adjacent rationals (~2.17e-19) is ~1000x smaller than a float64 ULP near 1.0 (~2.22e-16), causing mathematically proven collisions. The result is stored in `OfferOutput.Price` (type `float64`) and exported to both JSON and Parquet.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:173-185` — live offer export path, calls `TransformOffer()` for each offer change entry
- `internal/transform/offer.go:49-66` — validates `Price.N > 0`, `Price.D > 0`, computes `outputPrice = float64(outputPriceN) / float64(outputPriceD)`
- `internal/transform/offer.go:79-101` — emits `Price: outputPrice` alongside `PriceN: outputPriceN`, `PriceD: outputPriceD` in `OfferOutput`
- `internal/transform/schema.go:261-275` — `OfferOutput.Price` is `float64`, `PriceN` is `int32`, `PriceD` is `int32`
- `internal/transform/schema_parquet.go:194-208` — Parquet schema mirrors JSON: `Price float64`, `PriceN int32`, `PriceD int32`
- `internal/transform/operation.go:409-419` — operations use `price.String()` → `strconv.ParseFloat()`, a different path but same float64 limitation

### Findings

**The collision is mathematically proven and confirmed numerically.** Two concrete int32 pairs — `(2147478646, 2147478647)` and `(2147478647, 2147478648)` — produce identical float64 values when divided. Near int32 max, consecutive rationals of the form `n/(n+1)` and `(n+1)/(n+2)` differ by ~1/n², which at n ≈ 2^31 is ~2.17e-19, far below the float64 ULP of ~2.22e-16 at magnitude 1.0.

**Severity downgraded from Critical to Low** for three reasons:
1. **Companion exact fields exist**: `price_n` (int32) and `price_d` (int32) are always emitted alongside `price` in both JSON and Parquet, preserving the exact on-chain rational.
2. **Extreme trigger values required**: Both numerator and denominator must be near int32 max (~2 billion) for the collision to occur. On the real Stellar DEX, offer prices with both components this large are extremely unusual.
3. **Inherent float64 limitation**: This is a well-known property of IEEE-754 representation, not a code logic error. The computation `float64(N)/float64(D)` does exactly what it says; the issue is the type choice for this field.

The finding is real — the exported `price` column contains incorrect (indistinguishable) values for distinct on-chain prices — but the practical impact is limited.

### PoC Guidance

- **Test file**: `internal/transform/offer_test.go`
- **Setup**: Construct two `ingest.Change` objects with `OfferEntry.Price` set to `{N: 2147478646, D: 2147478647}` and `{N: 2147478647, D: 2147478648}` respectively. Use a minimal valid `LedgerHeaderHistoryEntry`.
- **Steps**: Call `TransformOffer()` on each change. Extract the `Price` field from each `OfferOutput`.
- **Assertion**: Assert that `output1.PriceN != output2.PriceN` (distinct on-chain values) but `output1.Price == output2.Price` (collapsed float64 values). This demonstrates the scalar price field cannot distinguish these two distinct on-chain prices.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-16
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestOfferPriceFloat64CollidesDistinctRationals"
**Test Language**: Go

### Demonstration

The test constructs two offer entries with distinct on-chain prices `2147478646/2147478647` and `2147478647/2147478648`, runs both through `TransformOffer()`, and confirms that while `PriceN`/`PriceD` remain distinct, the exported `Price` float64 field is identical (`9.99999999534337602469e-01`) for both. This proves the scalar price column cannot distinguish these two valid on-chain prices.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestOfferPriceFloat64CollidesDistinctRationals demonstrates that two distinct
// on-chain Price rationals collapse to the same float64 scalar when both
// numerator and denominator are near int32 max.
func TestOfferPriceFloat64CollidesDistinctRationals(t *testing.T) {
	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{CloseTime: 1000},
			LedgerSeq: 10,
		},
	}

	// Two distinct on-chain prices near int32 max
	price1 := xdr.Price{N: 2147478646, D: 2147478647}
	price2 := xdr.Price{N: 2147478647, D: 2147478648}

	change1 := wrapOfferEntry(xdr.OfferEntry{
		SellerId: genericAccountID,
		OfferId:  1,
		Selling:  nativeAsset,
		Buying:   nativeAsset,
		Amount:   100,
		Price:    price1,
	}, 1)

	change2 := wrapOfferEntry(xdr.OfferEntry{
		SellerId: genericAccountID,
		OfferId:  2,
		Selling:  nativeAsset,
		Buying:   nativeAsset,
		Amount:   100,
		Price:    price2,
	}, 1)

	out1, err := TransformOffer(change1, header)
	if err != nil {
		t.Fatalf("TransformOffer for price1 failed: %v", err)
	}

	out2, err := TransformOffer(change2, header)
	if err != nil {
		t.Fatalf("TransformOffer for price2 failed: %v", err)
	}

	// Confirm the on-chain rational components are distinct
	if out1.PriceN == out2.PriceN && out1.PriceD == out2.PriceD {
		t.Fatal("precondition failed: the two prices have identical N/D, test is invalid")
	}

	// Demonstrate the bug: distinct rationals collapse to the same float64
	if out1.Price == out2.Price {
		t.Errorf("BUG CONFIRMED: distinct on-chain prices collapsed to same float64\n"+
			"  price1 rational: %d/%d  -> float64: %.20e\n"+
			"  price2 rational: %d/%d  -> float64: %.20e",
			out1.PriceN, out1.PriceD, out1.Price,
			out2.PriceN, out2.PriceD, out2.Price)
	} else {
		t.Logf("prices are distinct: %.20e vs %.20e", out1.Price, out2.Price)
	}
}
```

### Test Output

```
=== RUN   TestOfferPriceFloat64CollidesDistinctRationals
    data_integrity_poc_test.go:59: BUG CONFIRMED: distinct on-chain prices collapsed to same float64
          price1 rational: 2147478646/2147478647  -> float64: 9.99999999534337602469e-01
          price2 rational: 2147478647/2147478648  -> float64: 9.99999999534337602469e-01
--- FAIL: TestOfferPriceFloat64CollidesDistinctRationals (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.812s
```
