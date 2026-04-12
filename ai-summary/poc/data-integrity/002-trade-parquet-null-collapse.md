# H002: Trade Parquet Collapses Nullable Routing Fields Into Zeros

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Trade exports should preserve whether a row came from an offer book or a liquidity pool. For offer-book trades, `selling_liquidity_pool_id`, `liquidity_pool_fee`, `rounding_slippage`, and `seller_is_exact` should stay absent. For liquidity-pool trades, `selling_offer_id` should stay absent because there is no seller offer to join against.

## Mechanism

`TradeOutput` models these fields with `null.String`, `null.Int`, and `null.Bool`, but `TradeOutputParquet` replaces them with plain `string`, `int64`, and `bool`. `TradeOutput.ToParquet()` then serializes `.String`, `.Int64`, and `.Bool`, so absent JSON values become `""`, `0`, and `false` in Parquet, silently rewriting row semantics and making offer trades look like pool trades with zero-valued attributes (or vice versa).

## Trigger

1. Run `export_trades` with `--parquet-output` on a ledger containing both offer-book trades and liquidity-pool trades.
2. Inspect an offer-book row: JSON omits `selling_liquidity_pool_id` / `liquidity_pool_fee`, but Parquet contains empty-string / zero defaults.
3. Inspect a liquidity-pool row: JSON omits `selling_offer_id`, but Parquet contains `0`.

## Target Code

- `internal/transform/trade.go:80-99` — liquidity-pool-only fields are populated conditionally
- `internal/transform/trade.go:110-157` — offer trades populate `selling_offer_id` while liquidity-pool trades leave it null
- `internal/transform/schema.go:303-311` — JSON schema marks these fields nullable
- `internal/transform/schema_parquet.go:236-243` — Parquet schema makes the same fields non-nullable primitives
- `internal/transform/parquet_converter.go:248-274` — converter flattens nulls to primitive zero values
- `cmd/export_trades.go:68-70` — live Parquet export path for trades

## Evidence

The transformer clearly uses nullability to distinguish trade shapes: liquidity-pool trades set `SellingLiquidityPoolID`, `LiquidityPoolFee`, `RoundingSlippage`, and `SellerIsExact`, while offer trades set `SellingOfferID`. The Parquet schema removes that nullability entirely, and the converter extracts raw primitive zero values from the null wrappers, so downstream Parquet readers lose the original routing information even though JSON preserved it.

## Anti-Evidence

Some zero values are valid in-domain for rows where the field is present, such as `rounding_slippage = 0` on a liquidity-pool trade. The bug is therefore not "zero is always wrong"; it is that Parquet cannot distinguish "field absent" from "field present with zero/false/empty" on the affected trade shapes.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `TransformTrade()` in `trade.go` through the `TradeOutput` JSON schema in `schema.go`, the `TradeOutputParquet` Parquet schema in `schema_parquet.go`, and the `ToParquet()` converter in `parquet_converter.go`. Confirmed that six nullable fields (`SellingOfferID`, `BuyingOfferID`, `SellingLiquidityPoolID`, `LiquidityPoolFee`, `RoundingSlippage`, `SellerIsExact`) are modeled as `null.Int`/`null.String`/`null.Bool` in the JSON schema but as plain `int64`/`string`/`bool` in the Parquet schema. The converter extracts `.Int64`, `.String`, and `.Bool` directly from the null wrappers, yielding zero values (`0`, `""`, `false`) when `Valid == false`. The `guregu/null` library embeds `sql.NullInt64`/`sql.NullBool` whose zero-value inner fields are `0`/`false`.

### Code Paths Examined

- `internal/transform/trade.go:80-114` — LP branch sets `liquidityPoolID`, `outputPoolFee`, `roundingSlippageBips` via `null.IntFrom()`/`null.StringFrom()`; offer branch sets `outputSellingOfferID` via `null.IntFrom()`. Unset fields remain as zero-value `null.X{}` (Valid=false).
- `internal/transform/trade.go:164-253` — `extractClaimedOffers()` sets `sellerIsExact` only for PathPaymentStrictSend (false) and PathPaymentStrictReceive (true); for ManageBuyOffer/ManageSellOffer/CreatePassiveSellOffer it stays as zero-value `null.Bool{}`.
- `internal/transform/schema.go:303-311` — `TradeOutput` declares `SellingOfferID null.Int`, `SellingLiquidityPoolID null.String`, `LiquidityPoolFee null.Int`, `RoundingSlippage null.Int`, `SellerIsExact null.Bool`.
- `internal/transform/schema_parquet.go:236-243` — `TradeOutputParquet` declares the same fields as plain `int64`, `string`, `int64`, `int64`, `bool`.
- `internal/transform/parquet_converter.go:248-274` — `ToParquet()` extracts `.Int64`, `.String`, `.Bool` directly, collapsing null to zero values.
- `github.com/guregu/null@v4.0.0/bool.go:14-16` and `int.go:15-17` — Confirmed `null.Bool` wraps `sql.NullBool{Bool: false, Valid: false}` and `null.Int` wraps `sql.NullInt64{Int64: 0, Valid: false}`.

### Findings

**Six fields lose null semantics in Parquet output:**

| Field | JSON (offer trade) | Parquet (offer trade) | JSON (LP trade) | Parquet (LP trade) |
|-------|--------------------|-----------------------|-----------------|--------------------|
| `selling_offer_id` | present (valid ID) | valid ID | null (absent) | `0` |
| `selling_liquidity_pool_id` | null (absent) | `""` | present (pool hex) | pool hex |
| `liquidity_pool_fee` | null (absent) | `0` | present (e.g. 30) | 30 |
| `rounding_slippage` | null (absent) | `0` | present (incl. valid 0) | value (incl. valid 0) |
| `seller_is_exact` | null (absent) for manage ops | `false` | null or valid | value or `false` |
| `buying_offer_id` | always set | always set | always set | always set |

**Most impactful collision — `seller_is_exact`:** ManageBuyOffer and ManageSellOffer trades get `seller_is_exact = false` in Parquet (collapsed from null), which is indistinguishable from PathPaymentStrictSend's legitimate `false`. The `trade_type` field (1=offer, 2=LP) does NOT disambiguate here because all three operation types produce trade_type=1 offer trades.

**`rounding_slippage` ambiguity:** LP trades with genuine `rounding_slippage = 0` are indistinguishable from offer trades where the field should be absent. Downstream queries like "find LP trades with zero rounding slippage" would incorrectly include all offer trades.

While `trade_type` partially mitigates some cases (you can filter by trade_type to know LP fields are irrelevant for type=1), it cannot distinguish null from zero within the same trade type, and `seller_is_exact` crosses trade types within type=1.

### PoC Guidance

- **Test file**: `internal/transform/trade_test.go` (append new test case)
- **Setup**: Create two `TradeOutput` values — one for an offer-book trade (LP fields null) and one for an LP trade (selling_offer_id null, seller_is_exact null). Call `ToParquet()` on both.
- **Steps**: (1) Construct a `TradeOutput` with `SellingLiquidityPoolID: null.String{}` (not set) and `SellingOfferID: null.IntFrom(12345)`. Call `ToParquet()`. (2) Construct a `TradeOutput` with `SellingLiquidityPoolID: null.StringFrom("abc")` and `SellingOfferID: null.Int{}` (not set). Call `ToParquet()`. (3) Construct a `TradeOutput` with `SellerIsExact: null.Bool{}` (ManageBuyOffer — should be absent). Call `ToParquet()`.
- **Assertion**: Assert that the offer-trade Parquet has `SellingLiquidityPoolID == ""` and `LiquidityPoolFee == 0` (demonstrating collapse). Assert that the LP-trade Parquet has `SellingOfferID == 0` (demonstrating collapse). Assert that the ManageBuyOffer Parquet has `SellerIsExact == false` — same as PathPaymentStrictSend's `null.BoolFrom(false)` after conversion, proving semantic collision.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTradeParquetNullCollapse"
**Test Language**: Go

### Demonstration

The test constructs four trade scenarios: (1) an offer-book trade with null LP fields, (2) an LP trade with null selling_offer_id, (3) a ManageBuyOffer vs PathPaymentStrictSend `seller_is_exact` collision, and (4) an LP trade with `rounding_slippage=0` vs an offer trade with absent slippage. In all cases, `ToParquet()` collapses null wrappers to zero values, making semantically distinct records indistinguishable in Parquet output.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/guregu/null"
)

// TestTradeParquetNullCollapse demonstrates that TradeOutput.ToParquet() collapses
// null.Int/null.String/null.Bool fields to zero values, making it impossible to
// distinguish "field absent" from "field present with zero/false/empty" in Parquet.
func TestTradeParquetNullCollapse(t *testing.T) {
	// === Case 1: Offer-book trade — LP fields should be absent ===
	offerTrade := TradeOutput{
		SellingOfferID:         null.IntFrom(12345),
		BuyingOfferID:          null.IntFrom(67890),
		SellingLiquidityPoolID: null.String{}, // absent — not an LP trade
		LiquidityPoolFee:       null.Int{},    // absent
		RoundingSlippage:       null.Int{},    // absent
		SellerIsExact:          null.Bool{},   // absent — ManageSellOffer has no exact semantics
		TradeType:              1,             // offer trade
	}

	offerParquet := offerTrade.ToParquet().(TradeOutputParquet)

	// These fields are absent in JSON (Valid=false), but Parquet collapses them to zero values
	if offerParquet.SellingLiquidityPoolID != "" {
		t.Errorf("Expected SellingLiquidityPoolID to collapse to empty string, got %q", offerParquet.SellingLiquidityPoolID)
	}
	if offerParquet.LiquidityPoolFee != 0 {
		t.Errorf("Expected LiquidityPoolFee to collapse to 0, got %d", offerParquet.LiquidityPoolFee)
	}
	if offerParquet.RoundingSlippage != 0 {
		t.Errorf("Expected RoundingSlippage to collapse to 0, got %d", offerParquet.RoundingSlippage)
	}
	if offerParquet.SellerIsExact != false {
		t.Errorf("Expected SellerIsExact to collapse to false, got %v", offerParquet.SellerIsExact)
	}

	t.Log("CONFIRMED: Offer-book trade LP fields collapsed to zero values in Parquet")

	// === Case 2: LP trade — SellingOfferID should be absent ===
	lpTrade := TradeOutput{
		SellingOfferID:         null.Int{},                       // absent — not an offer trade
		BuyingOfferID:          null.IntFrom(99999),
		SellingLiquidityPoolID: null.StringFrom("abc123poolhex"), // present
		LiquidityPoolFee:       null.IntFrom(30),                // present
		RoundingSlippage:       null.IntFrom(5),                 // present
		SellerIsExact:          null.Bool{},                      // absent for LP trades
		TradeType:              2,                                // LP trade
	}

	lpParquet := lpTrade.ToParquet().(TradeOutputParquet)

	// SellingOfferID is absent (null) but Parquet shows 0
	if lpParquet.SellingOfferID != 0 {
		t.Errorf("Expected SellingOfferID to collapse to 0, got %d", lpParquet.SellingOfferID)
	}

	t.Log("CONFIRMED: LP trade SellingOfferID collapsed to 0 in Parquet")

	// === Case 3: Semantic collision — ManageBuyOffer vs PathPaymentStrictSend ===
	// ManageBuyOffer: SellerIsExact is absent (null)
	manageBuyTrade := TradeOutput{
		SellingOfferID: null.IntFrom(11111),
		BuyingOfferID:  null.IntFrom(22222),
		SellerIsExact:  null.Bool{}, // absent — not applicable for ManageBuyOffer
		TradeType:      1,
	}

	// PathPaymentStrictSend: SellerIsExact is explicitly false
	strictSendTrade := TradeOutput{
		SellingOfferID: null.IntFrom(33333),
		BuyingOfferID:  null.IntFrom(44444),
		SellerIsExact:  null.BoolFrom(false), // explicitly false — valid value
		TradeType:      1,
	}

	manageBuyParquet := manageBuyTrade.ToParquet().(TradeOutputParquet)
	strictSendParquet := strictSendTrade.ToParquet().(TradeOutputParquet)

	// Both produce SellerIsExact=false in Parquet, but they mean different things:
	// - ManageBuyOffer: field should be absent (null)
	// - PathPaymentStrictSend: field is explicitly false
	if manageBuyParquet.SellerIsExact != strictSendParquet.SellerIsExact {
		t.Errorf("Expected semantic collision but values differ: manage=%v strict_send=%v",
			manageBuyParquet.SellerIsExact, strictSendParquet.SellerIsExact)
	}

	// Verify the source data IS distinguishable (Valid flag differs)
	if manageBuyTrade.SellerIsExact.Valid == strictSendTrade.SellerIsExact.Valid {
		t.Errorf("Expected source null.Bool Valid flags to differ, both are %v", manageBuyTrade.SellerIsExact.Valid)
	}

	t.Log("CONFIRMED: ManageBuyOffer (null) and PathPaymentStrictSend (false) both produce SellerIsExact=false in Parquet — semantic collision")

	// === Case 4: LP trade with rounding_slippage=0 vs offer trade (absent) ===
	lpZeroSlippage := TradeOutput{
		SellingLiquidityPoolID: null.StringFrom("pool789"),
		LiquidityPoolFee:       null.IntFrom(30),
		RoundingSlippage:       null.IntFrom(0), // legitimate zero
		TradeType:              2,
	}

	offerNoSlippage := TradeOutput{
		SellingOfferID:   null.IntFrom(55555),
		RoundingSlippage: null.Int{}, // absent
		TradeType:        1,
	}

	lpZeroParquet := lpZeroSlippage.ToParquet().(TradeOutputParquet)
	offerNoParquet := offerNoSlippage.ToParquet().(TradeOutputParquet)

	// Both produce RoundingSlippage=0 in Parquet
	if lpZeroParquet.RoundingSlippage != offerNoParquet.RoundingSlippage {
		t.Errorf("Expected RoundingSlippage collision but values differ: lp=%d offer=%d",
			lpZeroParquet.RoundingSlippage, offerNoParquet.RoundingSlippage)
	}

	// Verify source data IS distinguishable
	if lpZeroSlippage.RoundingSlippage.Valid == offerNoSlippage.RoundingSlippage.Valid {
		t.Errorf("Expected source Valid flags to differ, both are %v", lpZeroSlippage.RoundingSlippage.Valid)
	}

	t.Log("CONFIRMED: LP trade (rounding_slippage=0) and offer trade (absent) both produce RoundingSlippage=0 in Parquet — semantic collision")
}
```

### Test Output

```
=== RUN   TestTradeParquetNullCollapse
    data_integrity_poc_test.go:40: CONFIRMED: Offer-book trade LP fields collapsed to zero values in Parquet
    data_integrity_poc_test.go:60: CONFIRMED: LP trade SellingOfferID collapsed to 0 in Parquet
    data_integrity_poc_test.go:95: CONFIRMED: ManageBuyOffer (null) and PathPaymentStrictSend (false) both produce SellerIsExact=false in Parquet — semantic collision
    data_integrity_poc_test.go:125: CONFIRMED: LP trade (rounding_slippage=0) and offer trade (absent) both produce RoundingSlippage=0 in Parquet — semantic collision
--- PASS: TestTradeParquetNullCollapse (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.792s
```
