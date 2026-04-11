# H003: Normalized-offer `counter_amount` is derived from a double-rounded float product

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`dim_offers.counter_amount` should be derived from the exact on-chain offer amount and exact rational price, then rounded once at the final `float64` boundary. A large offer with `Amount = 9007199254740993` stroops and `Price = 123456789/10000000` should export `counter_amount = 11119998978.73516`, which is the `float64` representation of the exact rational product.

## Mechanism

`TransformOffer()` first rounds the raw stroop amount into `offer.Amount float64` and the rational price into `offer.Price float64`. `extractDimOffer()` then multiplies those two rounded floats (`float64(offer.Amount) * offer.Price`) instead of multiplying the original stroop integer by `PriceN/PriceD`. That double-rounding compounds error: for the trigger above, the current code exports `11119998978.735159` while the exact-rational-then-float path exports `11119998978.73516`, a drift of about 19 stroops.

## Trigger

Process any normalized offer with a large amount plus a non-trivial rational price, for example:

1. `OfferEntry.Amount = 9007199254740993`
2. `Price.N = 123456789`
3. `Price.D = 10000000`

Under the current code:

- `offer.Amount` becomes `900719925.4740993`
- `offer.Price` becomes `12.3456789`
- `counter_amount` exports as `11119998978.735159`

But the exact rational calculation `float((Amount / 1e7) * (N / D))` yields `11119998978.73516`.

## Target Code

- `internal/transform/offer.go:44-66,79-101` — converts raw offer amount and price into float64 fields before normalization
- `internal/transform/offer_normalized.go:139-167` — multiplies `offer.Amount` and `offer.Price` floats to derive `CounterAmount`
- `internal/transform/schema.go:321-329` — `DimOffer.CounterAmount` has no exact companion field, so the drift is unrecoverable from the normalized row

## Evidence

A brute-force search over valid `int64` amounts and `int32/int32` price fractions produces concrete divergences between the current float-product path and an exact rational product converted once at the end. The trigger above is one such case, and the discrepancy is larger than a single stroop, so downstream orderbook analytics can misstate quote-side size even when no hash collision occurs.

## Anti-Evidence

The output schema is still `float64`, so extremely large products will always have some precision limits. This hypothesis is narrower and testable: it targets an avoidable extra error from multiplying two already-rounded floats even though the exact rational inputs (`Amount`, `PriceN`, `PriceD`) are available earlier in the transform path.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`extractDimOffer()` at `offer_normalized.go:166` computes `CounterAmount: float64(offer.Amount) * offer.Price`, multiplying two independently-rounded float64 values. The `OfferOutput` struct passed to this function contains exact integer price components (`PriceN int32`, `PriceD int32`) that are never used in the computation. Using `offer.Amount * float64(offer.PriceN) / float64(offer.PriceD)` instead eliminates the pre-rounding of price and produces a result identical to the exact rational computation, verified with a Go test program.

### Code Paths Examined

- `internal/transform/offer.go:63-65` — `offer.Price` computed as `float64(outputPriceN) / float64(outputPriceD)`, rounding the rational price to float64
- `internal/transform/offer.go:90` — `offer.Amount` computed via `utils.ConvertStroopValueToReal(outputAmount)` using `big.NewRat().Float64()`, best single-value conversion
- `internal/transform/offer_normalized.go:166` — `CounterAmount: float64(offer.Amount) * offer.Price` multiplies two pre-rounded floats
- `internal/transform/offer_normalized.go:139` — function signature receives `offer OfferOutput` which includes `PriceN`/`PriceD` fields (exact int32 values)
- `internal/utils/main.go:85-88` — `ConvertStroopValueToReal` uses `big.NewRat(int64, 1e7).Float64()`

### Findings

**Verified numerically** with a Go program using the hypothesis trigger values (`Amount = 9007199254740993`, `PriceN = 123456789`, `PriceD = 10000000`):

| Path | CounterAmount (float64 bits) | Value |
|------|------------------------------|-------|
| Current: `offer.Amount * offer.Price` | `4204b66dc015e19b` | 11119998978.73515892 |
| Exact rational → float64 once | `4204b66dc015e19c` | 11119998978.73516083 |
| Alternative: `offer.Amount * float64(PriceN) / float64(PriceD)` | `4204b66dc015e19c` | 11119998978.73516083 |

The current code is off by **1 ULP** (~14.6 stroops). The alternative computation using `offer.PriceN`/`offer.PriceD` (already available in the struct) produces the **exact same result** as the rational computation — 0 stroops of difference. The fix requires no restructuring; only line 166 of `offer_normalized.go` needs to change.

This matches the pattern of VIABLE finding success/data-transform/016 (claimable balance inline float64 rounding): a better conversion path exists and is readily accessible but is not used, producing an incorrect financial field value.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go` (append to existing PoC test file)
- **Setup**: Create two `ingest.Change` offer entries with `OfferEntry.Amount = 9007199254740993`, `Price.N = 123456789`, `Price.D = 10000000`, and valid asset/seller fields. Run through `TransformOfferNormalized()`.
- **Steps**: Extract `Offer.CounterAmount` from the result. Compute the exact rational product `big.Rat(Amount*PriceN, 1e7*PriceD).Float64()`.
- **Assertion**: Assert that `Offer.CounterAmount` equals the exact rational product. Currently it does not — the current code produces `4204b66dc015e19b` while the exact result is `4204b66dc015e19c` (1 ULP off, ~14.6 stroops).
- **Fix verification**: Change `offer_normalized.go:166` from `float64(offer.Amount) * offer.Price` to `offer.Amount * float64(offer.PriceN) / float64(offer.PriceD)` and verify the assertion passes.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestNormalizedOfferCounterAmountDoubleRoundsFloatProduct"
**Test Language**: Go

### Demonstration

The test constructs an offer with `Amount = 9007199254740993` stroops and `Price = 123456789/10000000`, runs it through the full `TransformOfferNormalized()` production code path, then compares the output `CounterAmount` against the exact rational product computed via `big.Rat`. The current code produces `CounterAmount` with float64 bits `4204b66dc015e19b` while the exact value has bits `4204b66dc015e19c` — a 1 ULP difference caused by multiplying two pre-rounded floats instead of using the exact integer price components already available in the `OfferOutput` struct.

### Test Body

```go
package transform

import (
	"math"
	"math/big"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestNormalizedOfferCounterAmountDoubleRoundsFloatProduct demonstrates that
// extractDimOffer computes CounterAmount by multiplying two pre-rounded float64
// values (offer.Amount * offer.Price) instead of using the exact rational inputs.
// This double-rounding produces an incorrect counter_amount for large offers.
func TestNormalizedOfferCounterAmountDoubleRoundsFloatProduct(t *testing.T) {
	// Trigger values from the hypothesis
	var amount xdr.Int64 = 9007199254740993
	var priceN xdr.Int32 = 123456789
	var priceD xdr.Int32 = 10000000

	// Construct an offer with these values
	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeOffer,
		Pre:        nil,
		Post: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: xdr.Uint32(100),
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: testAccount1ID,
					OfferId:  999,
					Selling:  nativeAsset,
					Buying:   ethAsset,
					Amount:   amount,
					Price: xdr.Price{
						N: priceN,
						D: priceD,
					},
					Flags: 0,
				},
			},
		},
	}

	result, err := TransformOfferNormalized(change, 100)
	if err != nil {
		t.Fatalf("TransformOfferNormalized failed: %v", err)
	}

	gotCounterAmount := result.Offer.CounterAmount

	// Compute the exact rational product: (Amount / 1e7) * (PriceN / PriceD)
	// = Amount * PriceN / (1e7 * PriceD)
	num := new(big.Int).Mul(big.NewInt(int64(amount)), big.NewInt(int64(priceN)))
	den := new(big.Int).Mul(big.NewInt(1e7), big.NewInt(int64(priceD)))
	exactRat := new(big.Rat).SetFrac(num, den)
	exactFloat, _ := exactRat.Float64()

	// Compare bit patterns to detect the 1-ULP drift
	gotBits := math.Float64bits(gotCounterAmount)
	exactBits := math.Float64bits(exactFloat)

	t.Logf("Amount (stroops): %d", amount)
	t.Logf("PriceN: %d, PriceD: %d", priceN, priceD)
	t.Logf("Current CounterAmount:  %.20f (bits: %016x)", gotCounterAmount, gotBits)
	t.Logf("Exact rational float64: %.20f (bits: %016x)", exactFloat, exactBits)
	t.Logf("ULP difference: %d", int64(exactBits)-int64(gotBits))

	// Assert output matches the exact rational value.
	// If this fails, the double-rounding bug is proven.
	if gotBits != exactBits {
		t.Errorf("CounterAmount corrupted by double-rounding: off by %d ULP(s)\n"+
			"  got  bits %016x = %.20f\n"+
			"  want bits %016x = %.20f",
			int64(exactBits)-int64(gotBits), gotBits, gotCounterAmount, exactBits, exactFloat)
	}
}
```

### Test Output

```
=== RUN   TestNormalizedOfferCounterAmountDoubleRoundsFloatProduct
    data_integrity_poc_test.go:65: Amount (stroops): 9007199254740993
    data_integrity_poc_test.go:66: PriceN: 123456789, PriceD: 10000000
    data_integrity_poc_test.go:67: Current CounterAmount:  11119998978.73515892028808593750 (bits: 4204b66dc015e19b)
    data_integrity_poc_test.go:68: Exact rational float64: 11119998978.73516082763671875000 (bits: 4204b66dc015e19c)
    data_integrity_poc_test.go:69: ULP difference: 1
    data_integrity_poc_test.go:72: CounterAmount corrupted by double-rounding: off by 1 ULP(s)
      got  bits 4204b66dc015e19b = 11119998978.73515892028808593750
      want bits 4204b66dc015e19c = 11119998978.73516082763671875000
--- FAIL: TestNormalizedOfferCounterAmountDoubleRoundsFloatProduct (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.783s
```
