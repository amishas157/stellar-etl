# 001: DimOfferID rounds away one-stroop offer updates

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformOfferNormalized` derives `DimOfferID` from `fmt.Sprintf("%d/%f/%f", offer.OfferID, offer.Amount, offer.Price)`. Because `%f` defaults to six decimal places, two distinct offer states that differ only at Stellar's seventh stroop digit can hash to the same ID, and the orderbook export then collapses those states into one `dim_offers` row.

## Root Cause

`TransformOffer` converts the on-chain offer amount from integer stroops to a `float64` with seven-decimal Stellar precision. `extractDimOffer` then serializes that float with default `%f` formatting before hashing, which discards the seventh decimal digit; `OrderbookParser.parseOrderbook` deduplicates `dim_offers` solely on that derived hash while still emitting every `fact_offer_events` row.

## Reproduction

Create two real `ingest.Change` offer entries with the same `OfferID`, assets, and price, but amounts `10000001` and `10000002` stroops. Running `TransformOfferNormalized` on both yields different `BaseAmount` values but the same `DimOfferID`, so a normal orderbook export silently treats the later state as if it were the first one.

## Affected Code

- `internal/transform/offer.go:13-102` — converts `OfferEntry.Amount` from stroops to `float64`
- `internal/transform/offer_normalized.go:138-168` — hashes `OfferID`, amount, and price using `%f`
- `internal/input/orderbooks.go:72-117` — emits one `dim_offers` row per unique `DimOfferID` but appends every event

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestDimOfferIDCollision_SingleStroopDifference`
- **Test language**: go
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `internal/transform/data_integrity_poc_test.go`
  3. Run: `go test ./internal/transform/... -run TestDimOfferIDCollision_SingleStroopDifference -v`
  4. Observe: both offer states produce the same `DimOfferID=4586906745675598928` instead of distinct IDs

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestDimOfferIDCollision_SingleStroopDifference demonstrates that two offer
// states differing by 1 stroop produce the same DimOfferID because
// extractDimOffer uses fmt.Sprintf("%f") which truncates at 6 decimal places,
// losing the 7th-digit stroop precision.
func TestDimOfferIDCollision_SingleStroopDifference(t *testing.T) {
	amount1 := xdr.Int64(10000001)
	amount2 := xdr.Int64(10000002)

	makeChange := func(amount xdr.Int64) ingest.Change {
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
						OfferId:  12345,
						Selling:  nativeAsset,
						Buying:   ethAsset,
						Amount:   amount,
						Price:    xdr.Price{N: 1, D: 1},
						Flags:    0,
					},
				},
			},
		}
	}

	change1 := makeChange(amount1)
	change2 := makeChange(amount2)

	result1, err := TransformOfferNormalized(change1, 100)
	if err != nil {
		t.Fatalf("TransformOfferNormalized(amount=%d) error: %v", amount1, err)
	}

	result2, err := TransformOfferNormalized(change2, 100)
	if err != nil {
		t.Fatalf("TransformOfferNormalized(amount=%d) error: %v", amount2, err)
	}

	if result1.Offer.DimOfferID == result2.Offer.DimOfferID {
		t.Errorf("DimOfferID collision: two distinct offer states (amount=%d vs amount=%d) produce the same DimOfferID=%d. The 7th stroop digit is lost by %%f formatting.",
			amount1, amount2, result1.Offer.DimOfferID)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Two distinct offer states with different amounts should produce different `DimOfferID` values and therefore distinct `dim_offers` rows.
- **Actual**: A one-stroop difference is rounded away during hash input formatting, so both states get the same `DimOfferID` and downstream orderbook history collapses them.

## Adversarial Review

1. Exercises claimed bug: YES — the test calls `TransformOfferNormalized` directly on real `ingest.Change` values and observes identical `DimOfferID` output for different on-chain amounts.
2. Realistic preconditions: YES — Stellar offer amounts are integer stroops, and partial fills or edits can change an offer by a single stroop.
3. Bug vs by-design: BUG — `FactOfferEvent.OfferInstanceID` and `OrderbookParser` deduplication both rely on `DimOfferID` representing a distinct offer state, not an approximation.
4. Final severity: High — this is structural orderbook corruption rather than direct monetary-field corruption.
5. In scope: YES — the exporter emits plausible but wrong historical orderbook data with no error.
6. Test correctness: CORRECT — the only changed input is the amount, and the failure is on the exported identifier, not on a helper or mock.
7. Alternative explanations: NONE — the collision is produced by the `%f` hash input in `extractDimOffer`, and the downstream dedupe path consumes that exact value.
8. Novelty: NOT ASSESSED — duplicate handling is performed by the orchestrator.

## Suggested Fix

Hash lossless offer-state fields instead of formatted floats: use the raw stroop amount and price rational (`PriceN`/`PriceD`), or otherwise serialize with precision that cannot collapse valid Stellar values.
