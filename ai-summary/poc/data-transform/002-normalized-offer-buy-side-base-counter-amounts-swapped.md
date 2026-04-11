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
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestBuySideNormalizedOfferBaseCounterAmountsSwapped"
**Test Language**: Go

### Contract Violation Evidence

The reviewer asked for proof that the current behavior violates a contract, not just differs from a preferred interpretation. The following evidence establishes the internal contract and demonstrates its violation:

**1. Schema-level naming contract (schema.go:320–345)**

The `DimOffer` struct comment states it "aligns with the BigQuery table dim_offers." It contains `BaseAmount` and `CounterAmount` fields. The linked `DimMarket` struct (also stated to align with BigQuery `dim_markets`) defines `BaseCode`/`BaseIssuer` and `CounterCode`/`CounterIssuer`. The `DimOffer.MarketID` foreign key links each offer to its canonical market. The parallel naming convention (`Base*`/`Counter*` across both structs) establishes that `BaseAmount` denominates the `BaseCode` asset and `CounterAmount` denominates the `CounterCode` asset.

**2. Self-consistency violation across offer directions**

Two economically equivalent offers (100 ETH ↔ 200 native) in the same market (ETH/native) produce inconsistent `BaseAmount` values:
- Sell-side (sells ETH): `BaseAmount = 100` (ETH quantity ✓)
- Buy-side (sells native): `BaseAmount = 200` (native quantity ✗)

A downstream query like `SELECT SUM(base_amount) FROM dim_offers WHERE market_id = X` would sum 100 ETH + 200 native = 300 units of mixed assets. This is provably wrong regardless of which interpretation one assigns to the field names.

**3. PR #109 original implementation (commit 8ca325c)**

Commit `8ca325c` ("Add orderbook transform (#109)") introduced `offer_normalized.go` and the normalized schema. The commit message says "Added transform for normalized offers" and the schema comments claim alignment with BigQuery tables. The original implementation unconditionally assigned `BaseAmount = offer.Amount` without branching on `action`, and the unit test locked in the resulting values. This is consistent with an implementation oversight — the `action` field was correctly computed but not used to adjust the amount assignment. The PR description's claim of schema consistency supports the intent that these fields should align with the market's canonical sides, making the omission a bug rather than a design choice.

### Demonstration

The test constructs two economically equivalent offers (100 ETH ↔ 200 native) in the same canonical market (ETH base, native counter) — one sell-side and one buy-side. It verifies that sell-side correctly maps `BaseAmount` to the ETH (base) quantity, then proves that buy-side maps `BaseAmount` to the native (counter) quantity instead. The inconsistency between identical markets depending on offer direction proves the contract violation: `BaseAmount` cannot be reliably aggregated or compared across offers in the same market.

### Test Body

```go
package transform

import (
	"math"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestBuySideNormalizedOfferBaseCounterAmountsSwapped demonstrates that the DimOffer
// struct violates the contract established by DimMarket: for buy-side offers,
// BaseAmount contains the counter-asset quantity and CounterAmount contains the
// base-asset quantity.
//
// Proof strategy: construct two economically equivalent offers in the SAME market
// (ETH/native) — one sell-side and one buy-side — and show that:
//   - sell-side correctly maps BaseAmount → ETH quantity, CounterAmount → native quantity
//   - buy-side SWAPS them: BaseAmount → native quantity, CounterAmount → ETH quantity
//
// This is a contract violation because DimOffer.MarketID links to a DimMarket where
// BaseCode="ETH" and CounterCode="native". The BaseAmount field in DimOffer must
// always denominate the market's base asset (ETH). Aggregating BaseAmount across
// offers in the same market produces a sum of mixed asset quantities.
func TestBuySideNormalizedOfferBaseCounterAmountsSwapped(t *testing.T) {
	const ledgerSeq uint32 = 100

	// === SELL-SIDE OFFER ===
	// Sells 100 ETH (base asset) to buy native (counter asset)
	// XDR price = buying/selling = native/ETH = 2/1
	// So: 100 ETH → 200 native
	sellSideInput := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeOffer,
		Pre:        nil,
		Post: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: xdr.Uint32(30715263),
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: testAccount1ID,
					OfferId:  1001,
					Selling:  ethAsset,    // base asset
					Buying:   nativeAsset, // counter asset
					Amount:   1000000000,  // 100 ETH in stroops
					Price:    xdr.Price{N: 2, D: 1}, // 2 native per ETH
					Flags:    0,
				},
			},
		},
	}

	// === BUY-SIDE OFFER ===
	// Sells 200 native (counter asset) to buy ETH (base asset)
	// XDR price = buying/selling = ETH/native = 1/2
	// So: 200 native → 100 ETH (economically equivalent to sell-side)
	buySideInput := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeOffer,
		Pre:        nil,
		Post: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: xdr.Uint32(30715263),
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: testAccount1ID,
					OfferId:  1002,
					Selling:  nativeAsset, // counter asset
					Buying:   ethAsset,    // base asset
					Amount:   2000000000,  // 200 native in stroops
					Price:    xdr.Price{N: 1, D: 2}, // 0.5 ETH per native
					Flags:    0,
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

	// --- Verify both offers map to the same market with ETH as base ---

	if sellResult.Market.BaseCode != "ETH" || sellResult.Market.CounterCode != "native" {
		t.Fatalf("sell-side market unexpected: base=%s counter=%s",
			sellResult.Market.BaseCode, sellResult.Market.CounterCode)
	}
	if buyResult.Market.BaseCode != "ETH" || buyResult.Market.CounterCode != "native" {
		t.Fatalf("buy-side market unexpected: base=%s counter=%s",
			buyResult.Market.BaseCode, buyResult.Market.CounterCode)
	}
	if sellResult.Market.ID != buyResult.Market.ID {
		t.Fatalf("offers should share the same market ID: sell=%d buy=%d",
			sellResult.Market.ID, buyResult.Market.ID)
	}

	// --- Verify action correctness ---

	if sellResult.Offer.Action != "s" {
		t.Fatalf("sell-side action should be 's', got %q", sellResult.Offer.Action)
	}
	if buyResult.Offer.Action != "b" {
		t.Fatalf("buy-side action should be 'b', got %q", buyResult.Offer.Action)
	}

	// --- Verify sell-side amounts are correct ---
	// Sell-side: selling 100 ETH (base) for 200 native (counter)
	// BaseAmount should be 100 (ETH quantity), CounterAmount should be 200 (native quantity)

	const tolerance = 1e-9
	if math.Abs(sellResult.Offer.BaseAmount-100.0) > tolerance {
		t.Errorf("sell-side BaseAmount: got %v, want 100 (ETH)", sellResult.Offer.BaseAmount)
	}
	if math.Abs(sellResult.Offer.CounterAmount-200.0) > tolerance {
		t.Errorf("sell-side CounterAmount: got %v, want 200 (native)", sellResult.Offer.CounterAmount)
	}
	t.Logf("SELL-SIDE (action=%q): BaseAmount=%.1f (ETH ✓), CounterAmount=%.1f (native ✓)",
		sellResult.Offer.Action, sellResult.Offer.BaseAmount, sellResult.Offer.CounterAmount)

	// --- Demonstrate the buy-side bug ---
	// Buy-side: selling 200 native (counter) to buy 100 ETH (base)
	// Economically equivalent: same 100 ETH ↔ 200 native exchange.
	//
	// CORRECT: BaseAmount=100 (ETH quantity), CounterAmount=200 (native quantity)
	// ACTUAL:  BaseAmount=200 (native quantity), CounterAmount=100 (ETH quantity) — SWAPPED
	//
	// This proves a contract violation: in the same market, BaseAmount means
	// "ETH quantity" for sell-side but "native quantity" for buy-side.

	buySideBaseIsWrong := math.Abs(buyResult.Offer.BaseAmount-200.0) < tolerance
	buySideCounterIsWrong := math.Abs(buyResult.Offer.CounterAmount-100.0) < tolerance

	if !buySideBaseIsWrong {
		t.Errorf("Expected buy-side BaseAmount=200 (native, demonstrating the swap), got %v",
			buyResult.Offer.BaseAmount)
	}
	if !buySideCounterIsWrong {
		t.Errorf("Expected buy-side CounterAmount=100 (ETH, demonstrating the swap), got %v",
			buyResult.Offer.CounterAmount)
	}

	if buySideBaseIsWrong && buySideCounterIsWrong {
		t.Logf("BUY-SIDE  (action=%q): BaseAmount=%.1f (native ✗), CounterAmount=%.1f (ETH ✗)",
			buyResult.Offer.Action, buyResult.Offer.BaseAmount, buyResult.Offer.CounterAmount)
		t.Log("")
		t.Log("CONTRACT VIOLATION DEMONSTRATED:")
		t.Log("  Market: ETH(base) / native(counter)")
		t.Logf("  Sell-side BaseAmount=%.1f correctly holds ETH (base) quantity", sellResult.Offer.BaseAmount)
		t.Logf("  Buy-side  BaseAmount=%.1f incorrectly holds native (counter) quantity", buyResult.Offer.BaseAmount)
		t.Log("  Aggregating BaseAmount across both offers sums 100 ETH + 200 native = mixed-asset total")
	}
}
```

### Test Output

```
=== RUN   TestBuySideNormalizedOfferBaseCounterAmountsSwapped
    data_integrity_poc_test.go:123: SELL-SIDE (action="s"): BaseAmount=100.0 (ETH ✓), CounterAmount=200.0 (native ✓)
    data_integrity_poc_test.go:149: BUY-SIDE  (action="b"): BaseAmount=200.0 (native ✗), CounterAmount=100.0 (ETH ✗)
    data_integrity_poc_test.go:151: 
    data_integrity_poc_test.go:152: CONTRACT VIOLATION DEMONSTRATED:
    data_integrity_poc_test.go:153:   Market: ETH(base) / native(counter)
    data_integrity_poc_test.go:154:   Sell-side BaseAmount=100.0 correctly holds ETH (base) quantity
    data_integrity_poc_test.go:155:   Buy-side  BaseAmount=200.0 incorrectly holds native (counter) quantity
    data_integrity_poc_test.go:156:   Aggregating BaseAmount across both offers sums 100 ETH + 200 native = mixed-asset total
--- PASS: TestBuySideNormalizedOfferBaseCounterAmountsSwapped (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.818s
```

### Revision Notes: Addressing Reviewer Concerns

**Re: Authoritative schema contract** — No external `stellar/voyager#18` schema or BigQuery DDL exists in this repo. However, the internal schema in `schema.go:320–345` IS the authoritative contract: each struct's GoDoc comment explicitly states it "aligns with the BigQuery table" (`dim_offers`, `dim_markets`). The `DimOffer.MarketID` foreign key links to `DimMarket`, and the parallel naming (`BaseAmount`↔`BaseCode`, `CounterAmount`↔`CounterCode`) establishes the field-to-asset correspondence within the codebase itself.

**Re: Internal self-consistency invariant** — The revised PoC proves the contract violation through self-consistency: two economically equivalent offers (100 ETH ↔ 200 native) in the same market produce `BaseAmount = 100` (sell-side) and `BaseAmount = 200` (buy-side). Since both rows reference the same `DimMarket` where `BaseCode = "ETH"`, a `SUM(base_amount)` query produces 300 — mixing ETH and native quantities. This is provably wrong regardless of which interpretation one assigns to the field names.

**Re: PR #109 original-implementation evidence** — Commit `8ca325c` introduced the normalized-offer transform. The `action` field is correctly computed (branching on whether the selling asset equals the sorted base asset), but the amount assignment on lines 165–166 ignores this computation entirely — a classic "compute but don't use" oversight. The unit test then locked in the resulting values without noticing the semantic mismatch between `BaseCode = "ETH"` and `BaseAmount = 262.84` (native quantity). The PR description's claim of schema consistency supports the *intent* for these fields to correspond to canonical market sides, making the omission a bug in the implementation, not a deliberate design choice.
