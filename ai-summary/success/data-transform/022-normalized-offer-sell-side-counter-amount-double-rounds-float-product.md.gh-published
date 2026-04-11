# 022: Normalized-offer sell-side `counter_amount` double-rounds the float product

**Date**: 2026-04-11
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOfferNormalized()` exports sell-side `dim_offers.counter_amount` by multiplying two independently rounded `float64` values: the already-rounded offer amount and the already-rounded price. For large offers with non-trivial rational prices, that produces a different `float64` than converting the exact on-chain rational product once at the end, so the exported financial field is silently wrong.

## Root Cause

`TransformOffer()` converts `OfferEntry.Amount` to `OfferOutput.Amount float64` and `OfferEntry.Price` to `OfferOutput.Price float64`. `extractDimOffer()` then computes sell-side `CounterAmount` as `float64(offer.Amount) * offer.Price`, even though the exact integer/rational inputs still exist earlier in the transform path. That second rounding step loses one ULP for the reproduced case.

## Reproduction

Process a normal sell-side offer whose selling asset is the canonical base asset, with:

- `OfferEntry.Amount = 9007199254740993`
- `Price.N = 123456789`
- `Price.D = 10000000`
- `Selling = ethAsset`
- `Buying = nativeAsset`

The normalized row returns `action = "s"` and exports `counter_amount` with bits `4204b66dc015e19b`, while the exact rational product `(Amount * PriceN) / (1e7 * PriceD)` converted once to `float64` has bits `4204b66dc015e19c`.

## Affected Code

- `internal/transform/offer.go:TransformOffer:44-65,79-101` — converts raw offer amount and price into `OfferOutput` float fields
- `internal/transform/offer_normalized.go:TransformOfferNormalized:19-46` — routes normalized offers through `TransformOffer()`
- `internal/transform/offer_normalized.go:extractDimOffer:139-167` — computes sell-side `CounterAmount` from `offer.Amount * offer.Price`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestNormalizedOfferCounterAmountDoubleRoundsFloatProduct`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

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

func makeNormalizedOfferPOCChange(selling, buying xdr.Asset, amount xdr.Int64, priceN, priceD xdr.Int32) ingest.Change {
	return ingest.Change{
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
					Selling:  selling,
					Buying:   buying,
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
}

func exactOfferProduct(amount xdr.Int64, priceN, priceD xdr.Int32) float64 {
	num := new(big.Int).Mul(big.NewInt(int64(amount)), big.NewInt(int64(priceN)))
	den := new(big.Int).Mul(big.NewInt(1e7), big.NewInt(int64(priceD)))
	value, _ := new(big.Rat).SetFrac(num, den).Float64()
	return value
}

func TestNormalizedOfferCounterAmountDoubleRoundsFloatProduct(t *testing.T) {
	var amount xdr.Int64 = 9007199254740993
	var priceN xdr.Int32 = 123456789
	var priceD xdr.Int32 = 10000000

	result, err := TransformOfferNormalized(makeNormalizedOfferPOCChange(
		ethAsset,
		nativeAsset,
		amount,
		priceN,
		priceD,
	), 100)
	if err != nil {
		t.Fatalf("TransformOfferNormalized failed: %v", err)
	}

	if result.Offer.Action != "s" {
		t.Fatalf("expected sell-side normalized offer, got action %q", result.Offer.Action)
	}

	gotBits := math.Float64bits(result.Offer.CounterAmount)
	want := exactOfferProduct(amount, priceN, priceD)
	wantBits := math.Float64bits(want)

	t.Logf("counter_amount bits: %016x", gotBits)
	t.Logf("exact product bits: %016x", wantBits)
	t.Logf("counter_amount: %.20f", result.Offer.CounterAmount)
	t.Logf("exact product: %.20f", want)

	if gotBits != wantBits {
		t.Fatalf("counter_amount differs from exact rational product: got %016x want %016x", gotBits, wantBits)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `counter_amount` should equal the nearest `float64` to the exact on-chain rational product.
- **Actual**: `counter_amount` is one ULP lower because the code multiplies two pre-rounded floats instead of converting the exact product once.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC forces a sell-side row (`action = "s"`) and hits the exact `CounterAmount: offer.Amount * offer.Price` path.
2. Realistic preconditions: YES — the reproduced values are valid `xdr.Int64`/`xdr.Int32` offer fields and can occur in normal offer processing.
3. Bug vs by-design: BUG — the schema stores a `float64`, but the current path adds avoidable extra error beyond the best available `float64` result from the same on-chain inputs.
4. Final severity: Critical — `counter_amount` is a financial/orderbook amount field and the exported numeric value is wrong.
5. In scope: YES — this is silent data corruption in `internal/transform/`.
6. Test correctness: CORRECT — the test uses the production transform path, no mocks, and derives the expected value directly from exact XDR integers.
7. Alternative explanations: The original hypothesis trigger also overlapped the separate buy-side base/counter swap bug, but the final PoC removes that confounder by reproducing the mismatch on a sell-side row.
8. Novelty: NOVEL

## Suggested Fix

Preserve the raw `OfferEntry.Amount` integer through normalization, or compute normalized base/counter amounts directly from the raw `OfferEntry.Amount` and `OfferEntry.Price` rational before converting once to `float64`. Avoid deriving `counter_amount` from `offer.Amount * offer.Price` after both inputs have already been rounded.
