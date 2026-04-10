# H002: `DimOfferID` collapses distinct offer states at 6-decimal formatting

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Different historical states of the same offer should get different `dim_offer_id` values whenever either the amount or the price changes. An update from `amount = 1.0000001` to `amount = 1.0000002` should produce a new `dim_offer_id`, because those states represent different order-book rows.

## Mechanism

`extractDimOffer()` hashes `fmt.Sprintf("%d/%f/%f", offer.OfferID, offer.Amount, offer.Price)`. Go's default `%f` emits only 6 digits after the decimal point, so valid 7-digit stroop changes collapse to the same formatted string and therefore the same FNV hash.

## Trigger

Export normalized offers for the same `OfferID` across two ledgers where the amount changes by 1 stroop, or where the price differs only beyond the sixth decimal place.

## Target Code

- `internal/transform/offer.go:79-101` — converts the on-chain stroop amount into a 7-decimal `float64`
- `internal/transform/offer_normalized.go:139-168` — formats amount and price with `%f` before hashing
- `internal/transform/schema.go:321-329` — stores the derived `DimOfferID`

## Evidence

`OfferOutput.Amount` comes from `ConvertStroopValueToReal`, so one-stroop changes are represented at the seventh decimal place. `extractDimOffer()` discards that seventh digit before hashing, making collisions possible whenever two offer states only differ at that precision.

## Anti-Evidence

If every update changes amount or price by at least `0.000001`, the collision will not appear. The hash still distinguishes offers with different `OfferID` values.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced from XDR offer entry through `TransformOffer` (offer.go:90) where `ConvertStroopValueToReal` divides the stroop `Int64` by 1e7, producing float64 values with 7 decimal digits of stroop precision. This float64 flows into `extractDimOffer` (offer_normalized.go:140) where `fmt.Sprintf("%d/%f/%f", ...)` formats it with Go's default `%f` verb, which emits only 6 decimal places. The 7th digit is lost before the string is FNV-hashed into `DimOfferID`. Two offer states differing by 1–9 stroops (anywhere the 7th decimal digit differs) produce identical hash inputs and thus identical `DimOfferID` values.

### Code Paths Examined

- `internal/utils/main.go:85-88` — `ConvertStroopValueToReal` divides `xdr.Int64` by 10,000,000 via `big.NewRat().Float64()`, preserving 7-digit stroop precision in float64
- `internal/transform/offer.go:44,90` — `outputAmount` is `xdr.Int64` (offerEntry.Amount); line 90 converts to float64 via `ConvertStroopValueToReal`
- `internal/transform/offer_normalized.go:140` — `fmt.Sprintf("%d/%f/%f", offer.OfferID, offer.Amount, offer.Price)` uses `%f` which defaults to 6 decimal places, truncating the 7th stroop digit
- `internal/transform/offer_normalized.go:142-147` — FNV-64a hash of the truncated string becomes `DimOfferID`
- `internal/transform/offer.go:63-66` — `Price` is `float64(PriceN) / float64(PriceD)`, also subject to 7th-digit truncation by `%f`

### Findings

The bug is confirmed. Go's `%f` verb formats with 6 decimal places by default. Stellar uses 7-digit stroop precision (1 XLM = 10^7 stroops). Concrete example:

- 10,000,001 stroops → `ConvertStroopValueToReal` → `1.0000001` → `%f` → `"1.000000"`
- 10,000,009 stroops → `ConvertStroopValueToReal` → `1.0000009` → `%f` → `"1.000000"`

Both produce the same format string `"<id>/1.000000/<price>"` and the same FNV hash, so they get the same `DimOfferID`. Up to 10 consecutive stroop values can collide in each "bucket" of the 6th decimal place. The same truncation affects the Price field.

This causes distinct offer states to be incorrectly collapsed in the normalized orderbook dataset (`dim_offers` / `fact_offer_events`). Downstream analytics querying the historical orderbook would see fewer state changes than actually occurred, silently losing 1-stroop granularity updates.

### PoC Guidance

- **Test file**: `internal/transform/offer_normalized_test.go`
- **Setup**: Create two `ingest.Change` objects for the same `OfferID` with amounts differing by 1 stroop (e.g., `Amount: 10000001` and `Amount: 10000002`). Use identical selling/buying assets and price.
- **Steps**: Call `TransformOfferNormalized` on each change. Extract `DimOfferID` from both results.
- **Assertion**: Assert that the two `DimOfferID` values are different (`assert.NotEqual`). Currently this will FAIL because both produce the same hash, confirming the bug. The fix would be to use `%.7f` (or a stroop-aware integer format) in the `Sprintf` call at `offer_normalized.go:140`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestDimOfferIDCollision_SingleStroopDifference"
**Test Language**: Go

### Demonstration

The test creates two `ingest.Change` objects for the same OfferID (12345) with amounts differing by 1 stroop (10000001 vs 10000002). After `TransformOfferNormalized`, both produce the identical `DimOfferID=4586906745675598928` because `fmt.Sprintf("%d/%f/%f", ...)` truncates both `1.0000001` and `1.0000002` to `"1.000000"` before FNV hashing. This proves distinct offer states are silently collapsed into the same dimension key.

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
	// Two amounts that differ by 1 stroop:
	//   10000001 stroops → 1.0000001 XLM → %f → "1.000000"
	//   10000002 stroops → 1.0000002 XLM → %f → "1.000000"
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

	// The amounts are different, so DimOfferID should be different.
	// BUG: Both produce DimOfferID from hash of "12345/1.000000/1.000000"
	// because %f truncates to 6 decimal places, losing the 7th digit.
	if result1.Offer.DimOfferID == result2.Offer.DimOfferID {
		t.Errorf("DimOfferID collision: two distinct offer states (amount=%d vs amount=%d) "+
			"produce the same DimOfferID=%d. The 7th stroop digit is lost by %%f formatting.",
			amount1, amount2, result1.Offer.DimOfferID)
	}
}
```

### Test Output

```
=== RUN   TestDimOfferIDCollision_SingleStroopDifference
    data_integrity_poc_test.go:61: DimOfferID collision: two distinct offer states (amount=10000001 vs amount=10000002) produce the same DimOfferID=4586906745675598928. The 7th stroop digit is lost by %f formatting.
--- FAIL: TestDimOfferIDCollision_SingleStroopDifference (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.691s
```
