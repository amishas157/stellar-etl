# H001: Tiny offer prices collapse to numeric zero in operation details

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details.price` for `manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` should preserve the actual on-chain rational price as a nonzero numeric value whenever `Price.N > 0` and `Price.D > Price.N`. A valid order price like `1/2147483647` should therefore export as approximately `0.0000000004656613`, not `0`.

## Mechanism

`addPriceDetails()` converts the rational `xdr.Price` to a string via `price.String()` and then parses that rounded string back to `float64`. Upstream `xdr.Price.String()` uses `FloatString(7)`, so any legitimate price below `0.00000005` becomes the string `"0.0000000"` before parsing, and the ETL silently exports `details.price = 0` even though `details.price_r` still preserves the exact rational.

## Trigger

1. Export a ledger containing `manage_buy_offer`, `manage_sell_offer`, or `create_passive_sell_offer` with `Price{N:1, D:2147483647}` (or any other valid price below `5e-8`).
2. Inspect `history_operations.details.price`.
3. Observe that the ETL emits `0` while `details.price_r` still reports `{numerator: 1, denominator: 2147483647}`.

## Target Code

- `internal/transform/operation.go:addPriceDetails:409-419` — rounds `xdr.Price` through `price.String()` before `ParseFloat`
- `internal/transform/operation.go:extractOperationDetails:701-749` — live offer-operation branches that feed `details.price`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/price.go:String:7-10` — upstream formatter hard-codes 7 decimal places

## Evidence

`addPriceDetails()` never computes the rational directly from `price.N` and `price.D`; it trusts `price.String()`. The upstream formatter is explicitly `big.NewRat(...).FloatString(7)`, so values smaller than `0.00000005` round to `"0.0000000"` before the ETL parses them back into a numeric zero.

## Anti-Evidence

The exact rational survives in `details.price_r`, so downstream consumers that ignore `details.price` can reconstruct the truth. This does not save the exported numeric `price` field itself, which still becomes wrong while looking perfectly valid.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `addPriceDetails()` at `operation.go:409-421`, which calls `price.String()` (upstream `xdr/price.go:9` uses `big.NewRat(int64(p.N), int64(p.D)).FloatString(7)`) and then `strconv.ParseFloat(s, 64)`. Independently confirmed via a Go program that `Price{1, 2147483647}` produces `FloatString(7) = "0.0000000"` → `ParseFloat = 0`, while direct `float64(1)/float64(2147483647) = 4.656612875245797e-10`. The cutoff is at `D ≈ 20000000`: any price with `N/D < 5e-8` collapses to zero. Five call sites are affected across three offer operations and LP deposit min/max prices.

### Code Paths Examined

- `internal/transform/operation.go:409-421` — `addPriceDetails` calls `price.String()` → `strconv.ParseFloat`, setting `result["price"] = 0.0` for sub-5e-8 prices while `result["price_r"]` preserves the exact {N, D}
- `internal/transform/operation.go:701-711` — `ManageBuyOffer` branch calls `addPriceDetails(details, op.Price, "")`
- `internal/transform/operation.go:720-737` — `ManageSellOffer` branch calls `addPriceDetails(details, op.Price, "")`
- `internal/transform/operation.go:739-755` — `CreatePassiveSellOffer` branch calls `addPriceDetails(details, op.Price, "")`
- `internal/transform/operation.go:1008-1011` — LP deposit branch calls `addPriceDetails` for `op.MinPrice` and `op.MaxPrice`
- `go-stellar-sdk/xdr/price.go:9` — `Price.String()` returns `big.NewRat(int64(p.N), int64(p.D)).FloatString(7)`, hard-coding 7 decimal places

### Findings

The bug is confirmed with a concrete, reproducible trigger:

1. **Mechanism verified**: `FloatString(7)` produces exactly 7 decimal digits. For `Price{1, 2147483647}` (≈ 4.66e-10), the output is `"0.0000000"`. `strconv.ParseFloat("0.0000000", 64)` returns `0.0`.

2. **Threshold established**: Any `xdr.Price` with `N/D < 5e-8` (approximately `D > 20000000` when `N=1`) is affected. The `int32` domain allows `D` up to `2147483647`, giving a wide range of valid on-chain prices that collapse to zero.

3. **Distinct from prior findings**:
   - **fail/025** (`offers.price` float64 conversion): That was about `TransformOffer()` using `float64(N)/float64(D)` — direct division preserves nonzero values. Different code path, different mechanism.
   - **success/001** (DimOfferID collision): That was about `%f` formatting in FNV hash for offer deduplication. Different field, different code path.
   - **success/002** (token-transfer float64): That was about stroop integer-to-float64 precision loss. Different entity, different mechanism (this is string-formatting truncation of a rational, not float64 precision loss).

4. **Five affected call sites** across `ManageBuyOffer`, `ManageSellOffer`, `CreatePassiveSellOffer`, and `LiquidityPoolDeposit` (min/max price).

5. **Semantic inconsistency**: The same row exports `details.price = 0` alongside `details.price_r = {numerator: 1, denominator: 2147483647}`, creating a contradictory record where one field says "free" and the sibling says "nonzero price."

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or a new `internal/transform/data_integrity_poc_test.go`)
- **Setup**: Create a mock `ManageSellOffer` operation with `xdr.Price{N: 1, D: 2147483647}` using the existing test utilities (`utils.CreateSampleTx` or direct `xdr.Operation` construction)
- **Steps**: Call `TransformOperation()` on the mock operation and extract `details["price"]` and `details["price_r"]` from the result
- **Assertion**: Assert that `details["price"].(float64) == 0.0` (demonstrating the bug) while `details["price_r"].(Price).Numerator == 1` and `details["price_r"].(Price).Denominator == 2147483647` (demonstrating the inconsistency). The fix would compute `float64(price.N) / float64(price.D)` directly instead of routing through `price.String()`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestOfferPriceRoundsToZero"
**Test Language**: Go

### Demonstration

The test calls `addPriceDetails()` — the production function at `operation.go:409-421` — with `xdr.Price{N: 1, D: 2147483647}` (a valid on-chain price ≈ 4.66e-10). The function routes through `price.String()` which uses `FloatString(7)`, truncating the value to `"0.0000000"`, then `strconv.ParseFloat` converts it to `0.0`. The test confirms that `result["price"]` is `0.0` while `result["price_r"]` correctly preserves `{1, 2147483647}`, proving the semantic inconsistency in exported data.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestOfferPriceRoundsToZero demonstrates that addPriceDetails() collapses
// tiny but valid on-chain prices to numeric zero. The xdr.Price.String()
// upstream formatter uses FloatString(7) (7 decimal digits), so any price
// with N/D < 5e-8 rounds to "0.0000000" before ParseFloat, producing 0.0.
// Meanwhile price_r correctly preserves the exact rational {N, D}.
//
// If this test PASSES, the bug is confirmed (POC_PASS).
func TestOfferPriceRoundsToZero(t *testing.T) {
	// A valid on-chain price: 1/2147483647 ≈ 4.66e-10
	tinyPrice := xdr.Price{N: 1, D: 2147483647}

	result := make(map[string]interface{})
	err := addPriceDetails(result, tinyPrice, "")
	if err != nil {
		t.Fatalf("addPriceDetails returned unexpected error: %v", err)
	}

	// Extract the exported numeric price
	priceVal, ok := result["price"].(float64)
	if !ok {
		t.Fatalf("result[\"price\"] is not float64: %T", result["price"])
	}

	// Extract the rational price
	priceR, ok := result["price_r"].(Price)
	if !ok {
		t.Fatalf("result[\"price_r\"] is not Price: %T", result["price_r"])
	}

	// The rational price_r correctly preserves the exact numerator and denominator
	if priceR.Numerator != 1 || priceR.Denominator != 2147483647 {
		t.Errorf("price_r should preserve the rational: got {%d, %d}, want {1, 2147483647}",
			priceR.Numerator, priceR.Denominator)
	}

	// BUG ASSERTION: The numeric price is zero even though the true value is ~4.66e-10.
	correctPrice := float64(1) / float64(2147483647)
	if priceVal != 0.0 {
		t.Errorf("expected price to be 0.0 (demonstrating the bug), got %e", priceVal)
	}

	// Log the inconsistency for clarity
	t.Logf("BUG DEMONSTRATED: addPriceDetails exported price=%v but price_r={%d, %d} (true value ≈ %e)",
		priceVal, priceR.Numerator, priceR.Denominator, correctPrice)
	t.Logf("The price field and price_r field are semantically contradictory in the same row")
}
```

### Test Output

```
=== RUN   TestOfferPriceRoundsToZero
    data_integrity_poc_test.go:51: BUG DEMONSTRATED: addPriceDetails exported price=0 but price_r={1, 2147483647} (true value ≈ 4.656613e-10)
    data_integrity_poc_test.go:53: The price field and price_r field are semantically contradictory in the same row
--- PASS: TestOfferPriceRoundsToZero (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.970s
```
