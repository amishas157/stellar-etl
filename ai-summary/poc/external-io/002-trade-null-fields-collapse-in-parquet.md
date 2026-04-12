# H002: Trade Parquet collapses absent pool metadata into real zero and false values

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Trade rows that do not have liquidity-pool or path-payment-only metadata should preserve those fields as null in Parquet. In particular, an orderbook trade with no `liquidity_pool_fee`, no `rounding_slippage`, and no `seller_is_exact` value should remain distinguishable from a liquidity-pool or path-payment trade that legitimately has `liquidity_pool_fee=0`, `rounding_slippage=0`, or `seller_is_exact=false`.

## Mechanism

`TransformTrade()` intentionally leaves several fields as `null.Int`, `null.String`, or `null.Bool` depending on trade type, but `TradeOutput.ToParquet()` converts them to raw `int64`, `string`, and `bool` values using `.Int64`, `.String`, and `.Bool`. This erases nullability and causes unrelated trade types to collapse onto the same Parquet values, especially for `rounding_slippage=0` and `seller_is_exact=false`, which are both legitimate non-null outputs for real trades.

## Trigger

Run `export_trades --write-parquet` on a range containing:
1. a normal orderbook trade from `ManageSellOffer`/`ManageBuyOffer`, and
2. a liquidity-pool or path-payment trade whose `rounding_slippage` is `0` or whose `seller_is_exact` is `false`.

The JSON rows will differ (`null` versus concrete values), while the Parquet rows will converge on the same zero/false payload.

## Target Code

- `internal/transform/trade.go:80-157` — leaves `SellingOfferID`, `LiquidityPoolFee`, `RoundingSlippage`, and `SellerIsExact` nullable depending on trade type
- `internal/transform/parquet_converter.go:266-273` — flattens those nullable fields into plain `int64`/`bool` values
- `internal/transform/schema_parquet.go:236-243` — defines the affected Parquet columns as non-nullable scalars
- `internal/transform/trade_test.go:782-788` — existing fixture includes a pool/path trade with concrete `LiquidityPoolFee=30`, `RoundingSlippage=0`, `SellerIsExact=false`
- `internal/transform/trade_test.go:808-814` — existing fixture includes a pool/path trade with concrete non-null `RoundingSlippage` and `SellerIsExact=true`

## Evidence

The transform code clearly models these columns as optional because their meaning depends on the trade source. The Parquet converter then unconditionally dereferences the nullable wrappers, which makes missing values indistinguishable from explicit zeroes and `false`.

## Anti-Evidence

Some of these fields may be interpreted heuristically from `trade_type`, reducing ambiguity for specific analyses. But the export schema still claims to provide the columns directly, and their current Parquet values do not faithfully encode the JSON row semantics.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full data path from `TransformTrade()` through `ToParquet()` to the Parquet schema. Confirmed that `guregu/null` types (`null.Int`, `null.Bool`, `null.String`) with `.Valid=false` yield zero-values (0, false, "") when their inner fields are extracted in `ToParquet()`. The JSON marshaller correctly emits `null` for these fields, but Parquet output collapses them to type-default values. The most impactful ambiguity is `SellerIsExact`: both the null case (ManageBuyOffer orderbook trade) and the legitimate false case (PathPaymentStrictSend orderbook trade) produce `trade_type=1, seller_is_exact=false` in Parquet. No column in the Parquet schema disambiguates these — a consumer would need to join against operation data to recover the distinction.

### Code Paths Examined

- `internal/transform/schema.go:303-311` — `TradeOutput` defines 6 nullable fields using `null.Int`, `null.Bool`, `null.String` from `guregu/null`
- `internal/transform/trade.go:80-114` — `TransformTrade()` conditionally populates nullable fields: `outputPoolFee`, `roundingSlippageBips`, `liquidityPoolID` are only set for LP trades (type=2); `sellerIsExact` is only set for PathPaymentStrictSend (false) and PathPaymentStrictReceive (true); `outputSellingOfferID` is only set for orderbook trades (type=1)
- `internal/transform/trade.go:164-262` — `extractClaimedOffers()` sets `sellerIsExact` only for path payment operations (lines 227, 243), leaves it as zero-value `null.Bool{}` for ManageBuyOffer/ManageSellOffer/CreatePassiveSellOffer
- `internal/transform/parquet_converter.go:248-274` — `ToParquet()` extracts `.Int64`, `.Bool`, `.String` directly from null wrappers, discarding the `.Valid` flag
- `internal/transform/schema_parquet.go:236-243` — `TradeOutputParquet` defines all fields as non-nullable scalars (`int64`, `bool`, `string`)
- `internal/transform/parquet_converter.go:122,138,215,242` — Same pattern applied to `Sponsor.String` in accounts, account signers, trustlines, and offers — confirming this is systemic

### Findings

**Confirmed null-to-zero collapsing for 6 trade fields in Parquet:**

| Field | Null condition | Parquet value when null | Ambiguous with |
|-------|---------------|----------------------|----------------|
| `SellerIsExact` | ManageBuy/SellOffer, CreatePassive (type=1) | `false` | PathPaymentStrictSend (type=1, `false`) — **same trade_type, indistinguishable** |
| `RoundingSlippage` | All orderbook trades (type=1) | `0` | LP trade with 0-bips slippage (type=2) — distinguishable via trade_type |
| `SellingOfferID` | LP trades (type=2) | `0` | No real offer has ID=0, but 0 is still wrong data |
| `LiquidityPoolFee` | Orderbook trades (type=1) | `0` | Pools with fee=0 (theoretical, practice is 30bps) |
| `SellingLiquidityPoolID` | Orderbook trades (type=1) | `""` | Distinguishable — real pool IDs are non-empty |
| `BuyingOfferID` | Path payments without residual offer | `0` | Less impactful — synthetic TOID IDs are always > 0 |

**The most severe case is `SellerIsExact`**: both the null case (ManageBuyOffer orderbook trade) and the legitimate false case (PathPaymentStrictSend orderbook trade) produce `trade_type=1, seller_is_exact=false` in Parquet. No column in the Parquet schema disambiguates these — a consumer would need to join against operation data to recover the distinction.

**Systemic scope**: The same `.Field` extraction pattern is used for all `null.*` types across every `ToParquet()` method in the codebase (verified for `Sponsor` in accounts, signers, trustlines, and offers). Trade fields are the most impactful because their null values map to legitimate non-null values of the same type within the same `trade_type`.

### PoC Guidance

- **Test file**: `internal/transform/trade_test.go` (append new test case) or a standalone `internal/transform/parquet_converter_test.go`
- **Setup**: Create two `TradeOutput` structs:
  1. An orderbook trade from ManageBuyOffer: `SellerIsExact: null.Bool{}` (Valid=false), `RoundingSlippage: null.Int{}` (Valid=false)
  2. A PathPaymentStrictSend trade against orderbook: `SellerIsExact: null.BoolFrom(false)`, `RoundingSlippage: null.Int{}` (Valid=false)
- **Steps**: Call `ToParquet()` on both structs and compare the resulting `TradeOutputParquet` values
- **Assertion**: Assert that the two Parquet structs produce **identical** `SellerIsExact` values (`false`) despite the JSON structs marshalling differently (`null` vs `false`). Also assert `RoundingSlippage` is 0 for both despite one being null in JSON. This demonstrates the information loss.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTradeNullFieldsCollapseInParquet"
**Test Language**: Go

### Demonstration

The test constructs two `TradeOutput` structs: one simulating an orderbook trade (ManageBuyOffer) with null `SellerIsExact` and `RoundingSlippage`, and one simulating a PathPaymentStrictSend trade with concrete `false` and `0` values. JSON marshalling correctly distinguishes these (`null` vs `false`/`0`), but after `ToParquet()` conversion, both produce identical `SellerIsExact=false` and `RoundingSlippage=0` values, proving the information loss. The null-to-zero-value collapse is also confirmed for `SellingOfferID` and `LiquidityPoolFee`.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/guregu/null"
)

// TestTradeNullFieldsCollapseInParquet demonstrates that TradeOutput.ToParquet()
// collapses null fields into zero/false values, making them indistinguishable
// from legitimate concrete values. The most severe case is SellerIsExact: an
// orderbook trade from ManageBuyOffer (null) and a PathPaymentStrictSend trade
// (false) both produce seller_is_exact=false in Parquet despite differing in JSON.
func TestTradeNullFieldsCollapseInParquet(t *testing.T) {
	// Trade 1: orderbook trade from ManageBuyOffer — nullable fields are unset
	orderbookTrade := TradeOutput{
		SellingOfferID:   null.Int{},    // Valid=false (no selling offer concept here)
		BuyingOfferID:    null.IntFrom(42),
		RoundingSlippage: null.Int{},    // Valid=false (not applicable)
		SellerIsExact:    null.Bool{},   // Valid=false (not a path payment)
		LiquidityPoolFee: null.Int{},    // Valid=false (not an LP trade)
	}

	// Trade 2: PathPaymentStrictSend trade — has concrete false/zero values
	pathPaymentTrade := TradeOutput{
		SellingOfferID:   null.Int{},           // Valid=false (path payments don't have selling offers)
		BuyingOfferID:    null.IntFrom(99),
		RoundingSlippage: null.IntFrom(0),      // Valid=true, legitimately 0 slippage
		SellerIsExact:    null.BoolFrom(false),  // Valid=true, false because StrictSend
		LiquidityPoolFee: null.Int{},           // Valid=false (not an LP trade)
	}

	// Step 1: Verify JSON output preserves the distinction (null vs false/0)
	orderbookJSON, _ := json.Marshal(orderbookTrade)
	pathPaymentJSON, _ := json.Marshal(pathPaymentTrade)

	var orderbookMap, pathPaymentMap map[string]interface{}
	json.Unmarshal(orderbookJSON, &orderbookMap)
	json.Unmarshal(pathPaymentJSON, &pathPaymentMap)

	// In JSON, the orderbook trade has null seller_is_exact
	if orderbookMap["seller_is_exact"] != nil {
		t.Errorf("JSON: orderbook trade seller_is_exact should be null, got %v", orderbookMap["seller_is_exact"])
	}
	// In JSON, the path payment trade has false seller_is_exact
	if pathPaymentMap["seller_is_exact"] != false {
		t.Errorf("JSON: path payment trade seller_is_exact should be false, got %v", pathPaymentMap["seller_is_exact"])
	}
	// In JSON, the orderbook trade has null rounding_slippage
	if orderbookMap["rounding_slippage"] != nil {
		t.Errorf("JSON: orderbook trade rounding_slippage should be null, got %v", orderbookMap["rounding_slippage"])
	}
	// In JSON, the path payment trade has 0 rounding_slippage
	if pathPaymentMap["rounding_slippage"] != float64(0) {
		t.Errorf("JSON: path payment trade rounding_slippage should be 0, got %v", pathPaymentMap["rounding_slippage"])
	}

	t.Log("JSON output correctly distinguishes null from concrete values")

	// Step 2: Convert to Parquet and show the distinction is lost
	orderbookParquet := orderbookTrade.ToParquet().(TradeOutputParquet)
	pathPaymentParquet := pathPaymentTrade.ToParquet().(TradeOutputParquet)

	// BUG: Both produce SellerIsExact=false despite one being null in JSON
	if orderbookParquet.SellerIsExact != pathPaymentParquet.SellerIsExact {
		t.Errorf("Expected Parquet SellerIsExact to collapse (both false), but got orderbook=%v, pathpayment=%v",
			orderbookParquet.SellerIsExact, pathPaymentParquet.SellerIsExact)
	}
	t.Logf("BUG CONFIRMED: Parquet SellerIsExact: orderbook=%v (was null in JSON), pathpayment=%v (was false in JSON) — indistinguishable",
		orderbookParquet.SellerIsExact, pathPaymentParquet.SellerIsExact)

	// BUG: Both produce RoundingSlippage=0 despite one being null in JSON
	if orderbookParquet.RoundingSlippage != pathPaymentParquet.RoundingSlippage {
		t.Errorf("Expected Parquet RoundingSlippage to collapse (both 0), but got orderbook=%v, pathpayment=%v",
			orderbookParquet.RoundingSlippage, pathPaymentParquet.RoundingSlippage)
	}
	t.Logf("BUG CONFIRMED: Parquet RoundingSlippage: orderbook=%v (was null in JSON), pathpayment=%v (was 0 in JSON) — indistinguishable",
		orderbookParquet.RoundingSlippage, pathPaymentParquet.RoundingSlippage)

	// BUG: SellingOfferID also collapses null to 0
	if orderbookParquet.SellingOfferID != 0 {
		t.Errorf("Expected null SellingOfferID to collapse to 0, got %v", orderbookParquet.SellingOfferID)
	}
	t.Logf("BUG CONFIRMED: Parquet SellingOfferID: null collapsed to %v", orderbookParquet.SellingOfferID)

	// BUG: LiquidityPoolFee also collapses null to 0
	if orderbookParquet.LiquidityPoolFee != 0 {
		t.Errorf("Expected null LiquidityPoolFee to collapse to 0, got %v", orderbookParquet.LiquidityPoolFee)
	}
	t.Logf("BUG CONFIRMED: Parquet LiquidityPoolFee: null collapsed to %v", orderbookParquet.LiquidityPoolFee)
}
```

### Test Output

```
=== RUN   TestTradeNullFieldsCollapseInParquet
    data_integrity_poc_test.go:59: JSON output correctly distinguishes null from concrete values
    data_integrity_poc_test.go:70: BUG CONFIRMED: Parquet SellerIsExact: orderbook=false (was null in JSON), pathpayment=false (was false in JSON) — indistinguishable
    data_integrity_poc_test.go:78: BUG CONFIRMED: Parquet RoundingSlippage: orderbook=0 (was null in JSON), pathpayment=0 (was 0 in JSON) — indistinguishable
    data_integrity_poc_test.go:85: BUG CONFIRMED: Parquet SellingOfferID: null collapsed to 0
    data_integrity_poc_test.go:91: BUG CONFIRMED: Parquet LiquidityPoolFee: null collapsed to 0
--- PASS: TestTradeNullFieldsCollapseInParquet (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.873s
```
