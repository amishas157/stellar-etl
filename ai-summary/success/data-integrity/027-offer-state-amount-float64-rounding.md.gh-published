# 027: offer-state amount float64 rounding exports the wrong size

**Date**: 2026-04-14
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformOffer()` exports the authoritative `offers.amount` column by converting the exact XDR `OfferEntry.Amount` into a `float64`. Once the scaled offer amount is large enough, adjacent one-stroop values collapse to the same exported number even though the ledger entry itself remains exact.

This happens on the normal offer-state export path. For offer amounts `90071992547409930` and `90071992547409931`, `TransformOffer()` produces the same `Amount` value and `encoding/json` serializes both rows as `9007199254.740993`, so downstream order-book or reconciliation systems cannot tell which on-chain offer size was present.

## Root Cause

`TransformOffer()` keeps `OfferEntry.Amount` as an exact `xdr.Int64` until it constructs `OfferOutput`, then calls `utils.ConvertStroopValueToReal()` and stores the result in `OfferOutput.Amount`. That helper converts the rational stroop value through `big.Rat.Float64()`, rounding to the nearest IEEE-754 `float64` before JSON or Parquet serialization.

The offer schema makes that rounded `float64` the only exported amount field. `OfferOutputParquet.Amount` repeats the same lossy representation as Parquet `DOUBLE`, and normalized-offer generation reuses `offer.Amount`, so the precision loss can propagate into derived offer datasets too.

## Reproduction

Any normal export that includes an offer ledger entry with a sufficiently large `OfferEntry.Amount` can hit this path because `TransformOffer()` is the shared transformer for offer-state rows. The source amount is an ordinary XDR `Int64`, and the demonstrated trigger is only about `9,007,199,254.740993` units, well within Stellar's signed-64-bit amount range.

When two adjacent offer amounts above that threshold are transformed, the exported offer rows contain the same rounded `amount` float. Because no exact raw companion field is emitted, the one-stroop difference is permanently lost in both JSON and Parquet outputs.

## Affected Code

- `internal/transform/offer.go:TransformOffer:13-102` — reads exact `OfferEntry.Amount` and writes `OfferOutput.Amount` via `ConvertStroopValueToReal()`
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — rounds exact stroop integers to nearest `float64`
- `internal/transform/schema.go:OfferOutput:260-282` — exposes `offers.amount` only as `float64`
- `internal/transform/schema_parquet.go:OfferOutputParquet:193-216` — persists the same field as Parquet `DOUBLE`
- `internal/transform/parquet_converter.go:OfferOutput.ToParquet:222-244` — forwards the rounded JSON value directly into Parquet output
- `internal/transform/offer_normalized.go:extractDimOffer:138-165` — reuses the rounded offer amount in derived normalized-offer rows

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestOfferAmountRoundsLargeStroopValues`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./... && go test ./internal/transform/... -run '^TestOfferAmountRoundsLargeStroopValues$' -count=1 -v` after creating the target file with the body below.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestOfferAmountRoundsLargeStroopValues demonstrates that two offers
// differing by 1 stroop at large magnitudes produce identical float64
// Amount values after TransformOffer, proving precision loss in the
// exported financial field.
func TestOfferAmountRoundsLargeStroopValues(t *testing.T) {
	// Two stroop values that differ by 1, both above the float64
	// exact-integer boundary (~2^53 ≈ 9.007×10^15).
	const stroopA xdr.Int64 = 90071992547409930
	const stroopB xdr.Int64 = 90071992547409931

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	changeA := wrapOfferEntry(xdr.OfferEntry{
		SellerId: genericAccountID,
		OfferId:  1,
		Selling:  nativeAsset,
		Buying:   nativeAsset,
		Amount:   stroopA,
		Price:    xdr.Price{N: 1, D: 1},
	}, 1)

	changeB := wrapOfferEntry(xdr.OfferEntry{
		SellerId: genericAccountID,
		OfferId:  2,
		Selling:  nativeAsset,
		Buying:   nativeAsset,
		Amount:   stroopB,
		Price:    xdr.Price{N: 1, D: 1},
	}, 1)

	outputA, err := TransformOffer(changeA, header)
	if err != nil {
		t.Fatalf("TransformOffer(A) error: %v", err)
	}

	outputB, err := TransformOffer(changeB, header)
	if err != nil {
		t.Fatalf("TransformOffer(B) error: %v", err)
	}

	// The two stroop values differ by 1, so the exported amounts
	// SHOULD differ. If they are equal, precision was lost.
	if outputA.Amount == outputB.Amount {
		t.Errorf("Amount precision lost: two offers with stroops %d and %d "+
			"both exported Amount = %v", stroopA, stroopB, outputA.Amount)
	}

	// Also verify via JSON serialization that the values are indistinguishable.
	jsonA, _ := json.Marshal(outputA.Amount)
	jsonB, _ := json.Marshal(outputB.Amount)
	if string(jsonA) == string(jsonB) {
		t.Errorf("JSON serialization collapsed distinct stroops: "+
			"both serialize to %s", string(jsonA))
	}
}

// wrapOfferEntryForPoC is a helper identical to wrapOfferEntry; defined here
// so the PoC is self-contained if the existing helper ever moves.
func wrapOfferEntryForPoC(offerEntry xdr.OfferEntry, lastModified int) ingest.Change {
	return ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeOffer,
		Pre:        nil,
		Post: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: xdr.Uint32(lastModified),
			Data: xdr.LedgerEntryData{
				Type:  xdr.LedgerEntryTypeOffer,
				Offer: &offerEntry,
			},
		},
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Distinct on-chain offer sizes should remain distinguishable in exported `offers.amount`, e.g. `9007199254.7409930` vs `9007199254.7409931`.
- **Actual**: Both offer rows export the same floating-point `amount`, and JSON serializes both as `9007199254.740993`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `TransformOffer()` path with real `ingest.Change` offer entries and proves the serialized `amount` field collides when only the source stroop amount changes.
2. Realistic preconditions: YES — `OfferEntry.Amount` is an ordinary XDR `Int64`, and the demonstrated trigger is only about 9 billion units, well within the protocol type's allowed range for normal issued-asset offers.
3. Bug vs by-design: BUG — `offers.amount` is a canonical exported monetary column, not a display-only convenience field, and the schema provides no exact raw companion column that would let downstream consumers recover the lost stroop.
4. Final severity: Critical — this silently corrupts a monetary offer-size field in normal exports, so order-book analytics, reconciliation, or compliance consumers can read plausible but wrong values.
5. In scope: YES — this is concrete financial data corruption in repository-owned ETL code.
6. Test correctness: CORRECT — the final test uses real production types, invokes the live transformer, proves the exact source values differ by one stroop, and shows the collision survives JSON serialization.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Export offer amounts through an exact representation instead of making `float64` the only source of truth. The safest fix is to add an exact raw or decimal-string amount column for offers and normalized offers; if a rounded `float64` must remain for compatibility, treat it as a derived convenience field rather than the canonical exported amount.
