# H002: Buy-side normalized offers swap `base_amount` and `counter_amount`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`dim_offers.base_amount` should always describe the amount of the canonical base asset from `dim_markets`, and `counter_amount` should always describe the amount of the canonical counter asset. When `extractDimOffer()` marks an offer as `action = "b"` because the seller is offering the counter asset to buy the base asset, the exported amounts should flip accordingly: `base_amount` should be the bought quantity and `counter_amount` should be the sold quantity.

For the checked-in fixture in `offer_normalized_test.go`, the market is `ETH/native`, the offer sells `262.8450327` native, and the price implies buying `135.16473161502083` ETH. The correct normalized row should therefore be:

- `base_amount = 135.16473161502083`
- `counter_amount = 262.8450327`

## Mechanism

`extractDimOffer()` correctly derives `action = "b"` when `sellingAsset != assets[0]`, meaning the offer sells the counter asset rather than the base asset. But it still unconditionally writes `BaseAmount: offer.Amount` and `CounterAmount: float64(offer.Amount) * offer.Price`, so every buy-side row labels the sold counter quantity as `base_amount` and the bought base quantity as `counter_amount`.

## Trigger

Process any offer where the selling asset sorts after the buying asset in the canonical market ordering, for example the fixture in `internal/transform/offer_normalized_test.go`:

1. `Selling = native`
2. `Buying = ETH`
3. `Amount = 2628450327`
4. `Price = 920936891 / 1790879058`

The exported row becomes `action:"b", base_amount:262.8450327, counter_amount:135.16473161502083` even though the base asset is ETH and the counter asset is native.

## Target Code

- `internal/transform/offer_normalized.go:139-168` — computes `action` from sorted assets but does not use it when assigning `BaseAmount` / `CounterAmount`
- `internal/transform/schema.go:321-329` — `DimOffer` names the fields as canonical market-side amounts, not sold/bought amounts
- `internal/transform/offer_normalized_test.go:98-114` — current expected output already locks in the swapped values for a buy-side offer

## Evidence

The code already distinguishes buy- vs sell-side offers with the explicit `Action` field, so the function knows when `offer.Amount` is denominated in the counter asset rather than the base asset. The checked-in test fixture makes the mismatch concrete: `Market.BaseCode` is `ETH`, `Action` is `"b"`, but `BaseAmount` is still the sold native quantity.

## Anti-Evidence

The current unit test asserts the existing output, so this is not a regression that slipped past tests; it is a likely longstanding contract bug in the normalized-offer export. If downstream consumers intentionally reinterpret `base_amount` / `counter_amount` as sold/bought amounts instead of canonical market-side amounts, the field names themselves are misleading and reviewer confirmation of intended semantics will still be needed.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `TransformOfferNormalized` → `extractAssets` → `extractDimMarket` → `extractDimOffer`. The market construction in `extractDimMarket` correctly sorts assets alphabetically to establish a canonical base/counter pair (`assets[0]` = base, `assets[1]` = counter). The `extractDimOffer` function correctly re-sorts the same assets and derives `action = "b"` when the selling asset is the counter asset (not `assets[0]`). However, the amount assignment on lines 165–166 unconditionally sets `BaseAmount: offer.Amount` and `CounterAmount: float64(offer.Amount) * offer.Price`, ignoring the `action` flag entirely. For buy-side offers, `offer.Amount` is the quantity of the counter asset (the selling asset), not the base asset.

### Code Paths Examined

- `internal/transform/offer_normalized.go:16-57` — `TransformOfferNormalized` calls `extractAssets`, `extractDimMarket`, and `extractDimOffer` in sequence
- `internal/transform/offer_normalized.go:60-98` — `extractAssets` returns `(buyingAsset, sellingAsset)` with buying first; `offer.Amount` in the XDR always denominates the selling asset
- `internal/transform/offer_normalized.go:101-136` — `extractDimMarket` sorts assets alphabetically; `assets[0]` → base, `assets[1]` → counter; confirms `BaseCode = "ETH"`, `CounterCode = "native"` for the test fixture
- `internal/transform/offer_normalized.go:139-168` — `extractDimOffer` computes `action` correctly but does not branch on it for amount assignment; lines 165–166 always assign `BaseAmount = offer.Amount` (selling asset quantity) and `CounterAmount = offer.Amount * price` (buying asset quantity)
- `internal/transform/offer.go:44-90` — `TransformOffer` sets `Amount = ConvertStroopValueToReal(offerEntry.Amount)` (the selling asset amount in real units) and `Price = PriceN/PriceD` (buying per selling)
- `internal/transform/schema.go:320-330` — `DimOffer` struct fields `BaseAmount` and `CounterAmount` clearly correspond to the market's canonical base and counter assets
- `internal/transform/offer_normalized_test.go:92-124` — Test fixture: selling native, buying ETH, `Action: "b"`, `BaseCode: "ETH"`, but `BaseAmount: 262.84` (native quantity) — confirms the swap is locked in by the test

### Findings

The bug is confirmed: for every buy-side offer (`action = "b"`), the `BaseAmount` field contains the counter-asset quantity and the `CounterAmount` field contains the base-asset quantity. This is a direct financial data corruption because:

1. **`offer.Amount`** (XDR) is always the selling asset quantity. For sell-side offers the selling asset IS the base asset, so `BaseAmount = offer.Amount` is correct. For buy-side offers the selling asset is the counter asset, so `BaseAmount = offer.Amount` is wrong.

2. **`offer.Amount * offer.Price`** computes the buying asset quantity (since `Price = buying/selling`). For sell-side, buying = counter, so `CounterAmount = Amount * Price` is correct. For buy-side, buying = base, so this should be `BaseAmount`, not `CounterAmount`.

3. The unit test at line 111 confirms the swapped state: `BaseAmount: 262.8450327` (native quantity) while `Market.BaseCode` is `"ETH"` — these are semantically inconsistent.

4. Approximately half of all normalized offers in any production export (all buy-side offers) will have their base and counter amounts transposed. Any downstream analytics computing market depth, volume by asset, or VWAP from these columns would silently consume wrong values.

### PoC Guidance

- **Test file**: `internal/transform/offer_normalized_test.go`
- **Setup**: Use the existing `makeOfferNormalizedTestInput()` fixture (selling native, buying ETH) which produces `action = "b"`
- **Steps**: Call `TransformOfferNormalized(hardCodedInput, 100)` and inspect the `Offer` field
- **Assertion**: Assert that `result.Offer.BaseAmount` equals the ETH (base asset) quantity (`≈135.1647`) and `result.Offer.CounterAmount` equals the native (counter asset) quantity (`262.8450327`). The current test asserts the opposite, confirming the bug. The PoC should demonstrate the discrepancy by showing `result.Market.BaseCode == "ETH"` but `result.Offer.BaseAmount == 262.8450327` (which is the native/counter quantity, not the ETH/base quantity).

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestBuySideNormalizedOfferBaseCounterAmountsSwapped"
**Test Language**: Go

### Demonstration

The test constructs a buy-side offer (selling native, buying ETH) and calls `TransformOfferNormalized`. It verifies that the market correctly identifies ETH as the base asset and native as the counter asset, and that `action = "b"`. It then proves that `BaseAmount` contains the native (counter) quantity (262.8450327) instead of the ETH (base) quantity (135.1647), and `CounterAmount` contains the ETH (base) quantity instead of the native (counter) quantity — confirming that buy-side normalized offers have their base and counter amounts swapped.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestBuySideNormalizedOfferBaseCounterAmountsSwapped demonstrates that for buy-side
// normalized offers (action="b"), BaseAmount and CounterAmount are swapped relative
// to the canonical market definition.
func TestBuySideNormalizedOfferBaseCounterAmountsSwapped(t *testing.T) {
	// Setup: Use the same fixture as the existing test.
	// Offer sells native, buys ETH. Market sorts to ETH(base)/native(counter).
	// Since the seller is selling the counter asset (native), action = "b".
	input := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeOffer,
		Pre:        nil,
		Post: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: xdr.Uint32(30715263),
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: testAccount1ID,
					OfferId:  260678439,
					Selling:  nativeAsset,
					Buying:   ethAsset,
					Amount:   2628450327, // 262.8450327 native (the selling/counter asset)
					Price: xdr.Price{
						N: 920936891,
						D: 1790879058,
					},
					Flags: 2,
				},
			},
		},
	}

	result, err := TransformOfferNormalized(input, 100)
	if err != nil {
		t.Fatalf("TransformOfferNormalized failed: %v", err)
	}

	// Verify market canonical ordering: ETH is base, native is counter
	if result.Market.BaseCode != "ETH" {
		t.Fatalf("Expected market base code ETH, got %s", result.Market.BaseCode)
	}
	if result.Market.CounterCode != "native" {
		t.Fatalf("Expected market counter code native, got %s", result.Market.CounterCode)
	}

	// Verify action is buy-side
	if result.Offer.Action != "b" {
		t.Fatalf("Expected action 'b', got %s", result.Offer.Action)
	}

	// The offer sells 262.8450327 native and buys ETH at price 920936891/1790879058.
	// offer.Amount = 262.8450327 (native quantity, the selling asset)
	// offer.Amount * price = 135.16473161502083 (ETH quantity, the buying asset)
	//
	// Since ETH is the base asset and native is the counter asset:
	//   Correct BaseAmount  = ETH quantity  = 135.16473161502083
	//   Correct CounterAmount = native quantity = 262.8450327
	//
	// BUG: The code unconditionally assigns BaseAmount = offer.Amount (selling quantity)
	// and CounterAmount = offer.Amount * price (buying quantity), which is only correct
	// for sell-side offers. For buy-side offers, these are swapped.

	nativeQuantity := 262.8450327        // the selling (counter) asset amount
	ethQuantity := 135.16473161502083    // the buying (base) asset amount

	// Demonstrate the bug: BaseAmount contains the native (counter) quantity
	// instead of the ETH (base) quantity
	if result.Offer.BaseAmount == ethQuantity {
		t.Error("BaseAmount correctly holds ETH quantity — bug is not present (unexpected)")
	}
	if result.Offer.BaseAmount != nativeQuantity {
		t.Errorf("BaseAmount unexpected value: got %v, want %v (native quantity proving the swap)",
			result.Offer.BaseAmount, nativeQuantity)
	} else {
		t.Logf("BUG CONFIRMED: BaseAmount = %v (native/counter quantity), but base asset is ETH",
			result.Offer.BaseAmount)
	}

	// Demonstrate the bug: CounterAmount contains the ETH (base) quantity
	// instead of the native (counter) quantity
	if result.Offer.CounterAmount == nativeQuantity {
		t.Error("CounterAmount correctly holds native quantity — bug is not present (unexpected)")
	}
	if result.Offer.CounterAmount != ethQuantity {
		t.Errorf("CounterAmount unexpected value: got %v, want %v (ETH quantity proving the swap)",
			result.Offer.CounterAmount, ethQuantity)
	} else {
		t.Logf("BUG CONFIRMED: CounterAmount = %v (ETH/base quantity), but counter asset is native",
			result.Offer.CounterAmount)
	}
}
```

### Test Output

```
=== RUN   TestBuySideNormalizedOfferBaseCounterAmountsSwapped
    data_integrity_poc_test.go:83: BUG CONFIRMED: BaseAmount = 262.8450327 (native/counter quantity), but base asset is ETH
    data_integrity_poc_test.go:96: BUG CONFIRMED: CounterAmount = 135.16473161502083 (ETH/base quantity), but counter asset is native
--- PASS: TestBuySideNormalizedOfferBaseCounterAmountsSwapped (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.723s
```

---

## Final Review — Needs Revision

**Date**: 2026-04-11
**Final review by**: gpt-5.4, high

### What Needs Fixing

- The PoC proves the current implementation emits buy-side rows with `BaseAmount = offer.Amount` and `CounterAmount = offer.Amount * price`, but it does not prove that output violates the intended `dim_offers` contract.
- There is no authoritative schema or documentation in this repo showing that `dim_offers.base_amount` and `counter_amount` must always align with the canonical base/counter assets from `dim_markets`.
- The original normalized-offer implementation introduced this exact mapping and the original checked-in unit test expected the same `action:"b", base_amount: selling quantity, counter_amount: buying quantity` shape from day one.
- PR `#109` ("Add orderbook transform") described the transform as "consistent with the planned schemas for storing historical orderbooks", so the current behavior has non-trivial by-design evidence that the PoC does not address.

### Revision Instructions

- Find and cite an authoritative schema contract for the normalized orderbook tables (ideally the referenced `stellar/voyager#18` schema or equivalent downstream documentation) that explicitly defines `base_amount` and `counter_amount` as canonical market-side quantities.
- Rewrite the PoC framing so it proves a contract violation, not just current behavior. Right now the test only shows that buy-side rows differ from the hypothesis author's preferred interpretation.
- If no external schema exists, provide a concrete in-repo consumer or invariant showing that downstream code interprets `base_amount` as the canonical base-asset quantity and therefore miscomputes on buy-side rows.
- Address the original-implementation evidence directly: explain why the initial PR and initial unit test locked in the wrong semantics despite the PR claiming schema consistency.

### Checks Passed So Far

- Check 1: The code path exists and the PoC exercises `TransformOfferNormalized -> extractDimMarket -> extractDimOffer`.
- Check 2: The trigger is realistic; a normal buy-side offer (selling counter, buying base) reaches this path.
- Check 4: The PoC reproduces the current output, and the numeric amounts are consistent with the current implementation.
- Check 5: The finding is in scope if a contract violation can be established; it concerns exported orderbook correctness, not test-only behavior.
