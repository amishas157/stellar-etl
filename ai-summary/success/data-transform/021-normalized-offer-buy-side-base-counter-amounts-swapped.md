# 021: Normalized-offer buy-side base/counter amounts swapped

**Date**: 2026-04-11
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOfferNormalized()` emits a canonical `DimMarket` with sorted `BaseCode` / `CounterCode`, but `extractDimOffer()` always writes `BaseAmount = offer.Amount` and `CounterAmount = offer.Amount * offer.Price`. That is correct only when the offer sells the canonical base asset.

For buy-side offers, `offer.Amount` is the counter-asset quantity and `offer.Amount * offer.Price` is the base-asset quantity, so the exporter swaps the two monetary fields while still labeling the row with the same `market_id`.

## Root Cause

`extractDimOffer()` sorts the buying and selling assets, correctly derives `Action = "b"` when the seller is offering the canonical counter asset, and then ignores that result when assigning `BaseAmount` and `CounterAmount`.

Because `TransformOffer()` exposes `offer.Amount` as the selling-asset quantity and `offer.Price` as `buying / selling`, the unconditional assignment encodes sold/bought amounts instead of canonical base/counter amounts whenever `sellingAsset != baseAsset`.

## Reproduction

Construct two economically equivalent offers in the same `ETH/native` market:

1. A sell-side offer selling 100 ETH for 200 native
2. A buy-side offer selling 200 native for 100 ETH

Run both through `TransformOfferNormalized()`. Both rows point at the same market with `BaseCode = "ETH"` and `CounterCode = "native"`, but the buy-side row exports `BaseAmount = 200` and `CounterAmount = 100`, mixing counter-asset and base-asset units under the canonical column names.

## Affected Code

- `internal/transform/offer_normalized.go:101-168` — `extractDimMarket()` establishes canonical base/counter ordering, while `extractDimOffer()` ignores that ordering when assigning amounts.
- `internal/transform/offer.go:44-93` — `TransformOffer()` makes `offer.Amount` the selling-asset quantity and `offer.Price` the `buying/selling` ratio used in the swapped calculation.
- `internal/transform/schema.go:320-345` — `DimOffer` and `DimMarket` expose the canonical `base_*` / `counter_*` schema contract that the exported row violates.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestBuySideNormalizedOfferBaseCounterAmountsSwapped`
- **Test language**: go
- **How to run**: Create the target test file with the test body below, then run `go build ./...` and `go test ./internal/transform/... -run TestBuySideNormalizedOfferBaseCounterAmountsSwapped -v`.

### Test Body

```go
package transform

import (
	"math"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestBuySideNormalizedOfferBaseCounterAmountsSwapped(t *testing.T) {
	const ledgerSeq uint32 = 100
	const tolerance = 1e-9

	sellSideInput := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeOffer,
		Post: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: xdr.Uint32(30715263),
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: testAccount1ID,
					OfferId:  1001,
					Selling:  ethAsset,
					Buying:   nativeAsset,
					Amount:   1000000000,
					Price:    xdr.Price{N: 2, D: 1},
				},
			},
		},
	}

	buySideInput := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeOffer,
		Post: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: xdr.Uint32(30715263),
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: testAccount1ID,
					OfferId:  1002,
					Selling:  nativeAsset,
					Buying:   ethAsset,
					Amount:   2000000000,
					Price:    xdr.Price{N: 1, D: 2},
				},
			},
		},
	}

	sellResult, err := TransformOfferNormalized(sellSideInput, ledgerSeq)
	if err != nil {
		t.Fatalf("sell-side TransformOfferNormalized failed: %v", err)
	}

	buyResult, err := TransformOfferNormalized(buySideInput, ledgerSeq)
	if err != nil {
		t.Fatalf("buy-side TransformOfferNormalized failed: %v", err)
	}

	if sellResult.Market.BaseCode != "ETH" || sellResult.Market.CounterCode != "native" {
		t.Fatalf("sell-side market unexpected: base=%s counter=%s", sellResult.Market.BaseCode, sellResult.Market.CounterCode)
	}
	if buyResult.Market.BaseCode != "ETH" || buyResult.Market.CounterCode != "native" {
		t.Fatalf("buy-side market unexpected: base=%s counter=%s", buyResult.Market.BaseCode, buyResult.Market.CounterCode)
	}
	if sellResult.Market.ID != buyResult.Market.ID {
		t.Fatalf("offers should share the same market ID: sell=%d buy=%d", sellResult.Market.ID, buyResult.Market.ID)
	}

	if sellResult.Offer.Action != "s" {
		t.Fatalf("sell-side action should be 's', got %q", sellResult.Offer.Action)
	}
	if buyResult.Offer.Action != "b" {
		t.Fatalf("buy-side action should be 'b', got %q", buyResult.Offer.Action)
	}

	if math.Abs(sellResult.Offer.BaseAmount-100.0) > tolerance {
		t.Fatalf("sell-side BaseAmount: got %v, want 100 (ETH)", sellResult.Offer.BaseAmount)
	}
	if math.Abs(sellResult.Offer.CounterAmount-200.0) > tolerance {
		t.Fatalf("sell-side CounterAmount: got %v, want 200 (native)", sellResult.Offer.CounterAmount)
	}

	if math.Abs(buyResult.Offer.BaseAmount-200.0) > tolerance {
		t.Fatalf("expected buy-side BaseAmount=200 (counter/native, demonstrating the swap), got %v", buyResult.Offer.BaseAmount)
	}
	if math.Abs(buyResult.Offer.CounterAmount-100.0) > tolerance {
		t.Fatalf("expected buy-side CounterAmount=100 (base/ETH, demonstrating the swap), got %v", buyResult.Offer.CounterAmount)
	}
	if math.Abs(buyResult.Offer.BaseAmount-100.0) < tolerance {
		t.Fatalf("buy-side BaseAmount unexpectedly matched canonical base quantity")
	}
	if math.Abs(buyResult.Offer.CounterAmount-200.0) < tolerance {
		t.Fatalf("buy-side CounterAmount unexpectedly matched canonical counter quantity")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `base_amount` should always hold the quantity of `DimMarket.BaseCode`, and `counter_amount` should always hold the quantity of `DimMarket.CounterCode`, regardless of whether the offer is buy-side or sell-side.
- **Actual**: buy-side normalized offers export the sold counter-asset quantity as `base_amount` and the bought base-asset quantity as `counter_amount`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs the production `TransformOfferNormalized()` path with real offer entries and proves the same canonical market emits opposite asset units in `base_amount` depending on direction.
2. Realistic preconditions: YES — any offer whose selling asset sorts after its buying asset becomes `action = "b"` and hits the swapped assignment.
3. Bug vs by-design: BUG — the schema exposes canonical `base_*` / `counter_*` columns tied to `DimMarket`, while the implementation silently changes their units by direction.
4. Final severity: Critical — downstream monetary analytics over `base_amount` / `counter_amount` can sum mixed asset quantities inside one market.
5. In scope: YES — this is a concrete data-corruption issue in `internal/transform/`.
6. Test correctness: CORRECT — the test asserts transformed output values from two economically equivalent offers and fails unless the production code preserves the swapped mapping.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

In `extractDimOffer()`, branch on whether the selling asset is the canonical base asset. For sell-side rows keep the current mapping; for buy-side rows assign `BaseAmount = offer.Amount * offer.Price` and `CounterAmount = offer.Amount` so both fields always align with `DimMarket.BaseCode` and `DimMarket.CounterCode`.
