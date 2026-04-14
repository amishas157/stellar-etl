# H004: `offers.amount` rounds distinct large offer sizes together

**Date**: 2026-04-14
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Offer-state export should preserve the exact remaining offer amount represented by `OfferEntry.Amount`. Two valid offers whose remaining amounts differ by 1 stroop should export different `offers.amount` values.

## Mechanism

`TransformOffer()` takes the exact `Int64` `OfferEntry.Amount` and converts it to `float64` with `utils.ConvertStroopValueToReal()` before storing it in `OfferOutput.Amount`. At sufficiently large magnitudes, adjacent offer sizes collapse to the same plausible decimal value, so downstream order-book reconstruction sees a silently corrupted size even though the XDR source remained exact up to that final assignment.

## Trigger

Feed `TransformOffer()` two otherwise identical offer ledger entries whose `Amount` values are `90071992547409930` and `90071992547409931` stroops. Compare the resulting `OfferOutput.Amount` values after JSON export: both should serialize to the same rounded number despite the one-stroop difference on-chain.

## Target Code

- `internal/transform/offer.go:44-47` — reads exact `OfferEntry.Amount`
- `internal/transform/offer.go:79-101` — writes `OfferOutput.Amount` via `ConvertStroopValueToReal`
- `internal/transform/schema.go:260-282` — `OfferOutput.Amount` is the exported state column
- `internal/utils/main.go:84-87` — stroop conversion returns `float64`
- `.../xdr/xdr_generated.go:5427-5433` — `OfferEntry.Amount` is exact XDR `Int64`

## Evidence

The transform path keeps `OfferEntry.Amount` as an integer until the final row assembly, so the corruption is introduced entirely by the ETL's own type choice. This field is especially important because sibling normalized-offer logic already depends on `OfferOutput.Amount`, so the rounded value can contaminate both raw offer exports and derived order-book tables.

## Anti-Evidence

The bug requires unusually large outstanding offers; ordinary sizes still export distinctly. The existing offer-detail rounding finding only covers operation-detail maps, not the persisted offer-state table targeted here.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `OfferEntry.Amount` (XDR `Int64`) through `TransformOffer()` at `offer.go:44-90` into `OfferOutput.Amount` (schema `float64`). The conversion uses `utils.ConvertStroopValueToReal()` which calls `big.NewRat(int64(input), 10000000).Float64()`, returning the nearest IEEE-754 double. For stroop values exceeding ~2^53 (the float64 exact-integer boundary), adjacent values collapse to the same float64. Both JSON and Parquet exports carry this lossy value — `OfferOutputParquet.Amount` is also `float64`.

### Code Paths Examined

- `internal/transform/offer.go:44` — `outputAmount := offerEntry.Amount` reads exact `xdr.Int64`
- `internal/transform/offer.go:90` — `Amount: utils.ConvertStroopValueToReal(outputAmount)` converts to lossy `float64`
- `internal/utils/main.go:85-87` — `ConvertStroopValueToReal` uses `big.NewRat(...).Float64()`, rounds to nearest double
- `internal/transform/schema.go:271` — `Amount float64` in `OfferOutput` struct
- `internal/transform/schema_parquet.go` — `OfferOutputParquet.Amount` is `float64` (`type=DOUBLE`)
- `internal/transform/parquet_converter.go:234` — `Amount: oo.Amount` copies the already-rounded value directly
- `internal/transform/offer_normalized.go:140` — `extractDimOffer` uses rounded `offer.Amount` in hash input, compounding the issue into normalized offer deduplication

### Findings

1. **The bug is real.** `OfferOutput.Amount` is a financial field (remaining offer size on the DEX) stored as `float64`. For stroop values above ~9×10^15, adjacent values produce identical exports. The trigger threshold (~900 million in asset units) is realistic for custom Stellar assets where issuers routinely create supplies in the billions or trillions.

2. **Both JSON and Parquet are affected.** The Parquet converter copies `oo.Amount` without re-conversion, so both export formats carry the same rounded value.

3. **Downstream contamination.** The rounded `Amount` feeds into `extractDimOffer` (line 140 of `offer_normalized.go`), contaminating the `BaseAmount` and `CounterAmount` fields of the normalized offer table. This is downstream of but distinct from the already-confirmed `DimOfferID` hash collision (success/001), which was about `%f` formatting precision, not the source `float64` value itself.

4. **Same root cause pattern as success/002 (token transfer) and success/024 (change trust limit)**, but different entity, code path, and downstream impact. Each is a separate exportable field with its own consumers.

### PoC Guidance

- **Test file**: `internal/transform/offer_test.go`
- **Setup**: Construct two `ingest.Change` objects with `OfferEntry.Amount` values of `90071992547409930` and `90071992547409931` (differ by 1 stroop). Use identical seller, assets, price, and offer ID otherwise.
- **Steps**: Call `TransformOffer()` on each change. Compare the resulting `OfferOutput.Amount` values.
- **Assertion**: Assert that the two `Amount` values are NOT equal (this assertion will FAIL, proving the bug). Also verify via `json.Marshal` that both rows serialize to the same `"amount"` JSON number.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-14
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestOfferAmountRoundsLargeStroopValues"
**Test Language**: Go

### Demonstration

The test constructs two offer entries with stroop values 90071992547409930 and 90071992547409931 (differing by 1), runs both through `TransformOffer()`, and verifies that the resulting `OfferOutput.Amount` float64 values are identical — proving that the `float64` conversion silently collapses distinct financial values. JSON serialization confirms both export to the same number `9007199254.740993`.

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

### Test Output

```
=== RUN   TestOfferAmountRoundsLargeStroopValues
    data_integrity_poc_test.go:61: Amount precision lost: two offers with stroops 90071992547409930 and 90071992547409931 both exported Amount = 9.007199254740993e+09
    data_integrity_poc_test.go:69: JSON serialization collapsed distinct stroops: both serialize to 9007199254.740993
--- FAIL: TestOfferAmountRoundsLargeStroopValues (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.960s
```
