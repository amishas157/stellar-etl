# H003: Trade Parquet collapses null `rounding_slippage` into explicit `0`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Parquet trade rows should preserve the same three `rounding_slippage` states that the JSON transform can emit: null when the concept does not apply, `0` when a liquidity-pool trade had no rounding loss, and a positive or sentinel value when rounding slippage was measured. Order-book trades should not be exported as though they had exactly zero rounding slippage.

## Mechanism

`TradeOutput` models `RoundingSlippage` as `null.Int`, and the fixtures already show both null and explicit `0` as valid JSON-layer states. `TradeOutputParquet` narrows the field to required `int64`, and `TradeOutput.ToParquet()` writes `to.RoundingSlippage.Int64`, so null order-book trades and exact-zero liquidity-pool trades both serialize as `rounding_slippage = 0`.

## Trigger

Run `export_trades --write-parquet` on a ledger range containing both an order-book trade and a liquidity-pool trade whose computed `rounding_slippage` is exactly zero. The JSON rows remain distinguishable (`null` versus `0`), but the Parquet rows export the same numeric value `0` for both cases.

## Target Code

- `internal/transform/schema.go:TradeOutput:303-311` — JSON/output schema models `rounding_slippage` as `null.Int`
- `internal/transform/trade.go:80-105` — only liquidity-pool trades can populate `roundingSlippageBips`; order-book trades leave it null
- `internal/transform/trade.go:148-156` — transformed trades carry the nullable `RoundingSlippage` field into output rows
- `internal/transform/trade.go:350-399` — `roundingSlippage()` returns explicit numeric values, including real `0`
- `internal/transform/schema_parquet.go:TradeOutputParquet:236-243` — Parquet schema narrows `rounding_slippage` to required `int64`
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:266-273` — converter serializes `to.RoundingSlippage.Int64` without checking validity
- `internal/transform/trade_test.go:738-742,782-789` — fixtures show order-book trades with null slippage and liquidity-pool trades with explicit `null.IntFrom(0)`

## Evidence

The trade fixtures already establish the collision: the order-book examples at `trade_test.go:738-742` have no `RoundingSlippage` field set, while the liquidity-pool example at `trade_test.go:782-789` sets `RoundingSlippage: null.IntFrom(0)`. Because the Parquet converter writes `.Int64` into a required `int64` column, both rows become indistinguishable `0` in Parquet.

## Anti-Evidence

Downstream consumers can partially reconstruct applicability from `trade_type`, because the null state appears on order-book trades while explicit values occur on liquidity-pool trades. But the field itself is still corrupted: Parquet claims an exact numeric slippage value where the JSON export correctly says the metric was not applicable at all.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full path from `TransformTrade()` through `ToParquet()`. Order-book trades (`tradeType=1`) skip the liquidity-pool branch entirely (lines 85-109 of trade.go), leaving `roundingSlippageBips` as its zero-value `null.Int{}` (`Valid=false, Int64=0`). Liquidity-pool trades (`tradeType=2`) call `roundingSlippage()` which can return `null.IntFrom(0)` (`Valid=true, Int64=0`). The converter at line 272 of parquet_converter.go reads `.Int64` from both, producing `0` in both cases. The Parquet schema at line 242 of schema_parquet.go declares this as a required `int64`, so there is no way to represent null.

### Code Paths Examined

- `internal/transform/schema.go:309` — `RoundingSlippage null.Int` confirms nullable JSON-layer type
- `internal/transform/schema_parquet.go:242` — `RoundingSlippage int64` confirms required non-nullable Parquet type
- `internal/transform/parquet_converter.go:272` — `RoundingSlippage: to.RoundingSlippage.Int64` extracts raw int64 without validity check
- `internal/transform/trade.go:80-114` — order-book trades never assign `roundingSlippageBips`, leaving it as zero-value `null.Int{}`
- `internal/transform/trade.go:100-108` — liquidity-pool trades call `roundingSlippage()` which returns `null.IntFrom(int64(roundingSlippageBips))`
- `internal/transform/trade.go:350-397` — `roundingSlippage()` only handles `PathPaymentStrictReceive` and `PathPaymentStrictSend`, returning explicit `null.IntFrom()` values including 0
- `internal/transform/trade_test.go:738-742` — order-book fixture has no `RoundingSlippage` set (zero-value null.Int)
- `internal/transform/trade_test.go:787` — LP fixture has `RoundingSlippage: null.IntFrom(0)` (valid, explicit 0)

### Findings

This is the exact same class of bug as the confirmed `seller_is_exact` null collapse (success/004), applied to the `rounding_slippage` field. The JSON layer correctly distinguishes "not applicable" (`null.Int{}`) from "measured as zero" (`null.IntFrom(0)`), but `ToParquet()` destroys this distinction by reading `.Int64` into a required `int64` column. The bug is structurally identical and independently exploitable in trade export pipelines.

### PoC Guidance

- **Test file**: `internal/transform/trade_test.go` (or a dedicated `data_integrity_poc_test.go`)
- **Setup**: Use the existing `makeTradeTestInput()` fixture. Transform with `operationIndex=0` (manage-sell-offer, produces order-book trade) and `operationIndex=2` (path-payment-strict-send, produces LP trade with `RoundingSlippage: null.IntFrom(0)`).
- **Steps**: Call `TransformTrade()` for both operation indices. Verify JSON-layer preconditions: order-book trade has `RoundingSlippage.Valid == false`, LP trade has `RoundingSlippage.Valid == true && RoundingSlippage.Int64 == 0`. Then call `ToParquet()` on both.
- **Assertion**: Assert that `manageOfferParquet.RoundingSlippage != lpTradeParquet.RoundingSlippage` — this will fail, confirming the null collapse. Alternatively, assert there exists some mechanism to distinguish the two states in Parquet (there isn't).

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestParquetCollapsesNullRoundingSlippageToZero"
**Test Language**: Go

### Demonstration

The test constructs two `TradeOutput` structs — an order-book trade with `RoundingSlippage: null.Int{}` (Valid=false) and an LP trade with `RoundingSlippage: null.IntFrom(0)` (Valid=true, Int64=0). It verifies the JSON layer correctly distinguishes these as `null` vs `0`, then calls `ToParquet()` on both and confirms that both Parquet rows produce identical `rounding_slippage=0`, proving the null/zero distinction is destroyed during Parquet conversion.

### Test Body

```go
func TestParquetCollapsesNullRoundingSlippageToZero(t *testing.T) {
	// Order-book trade: RoundingSlippage is not applicable (null.Int{}, Valid=false)
	orderBookTrade := TradeOutput{
		Order:                0,
		LedgerClosedAt:       time.Now(),
		SellingAccountAddress: "GABC",
		SellingAssetType:     "credit_alphanum4",
		BuyingAccountAddress: "GDEF",
		BuyingAssetType:      "native",
		SellingOfferID:       null.IntFrom(12345),
		BuyingOfferID:        null.IntFrom(67890),
		HistoryOperationID:   100,
		TradeType:            1, // order-book
		RoundingSlippage:     null.Int{}, // NOT APPLICABLE — Valid=false, Int64=0
		SellerIsExact:        null.Bool{},
	}

	// LP trade: RoundingSlippage is explicitly zero (null.IntFrom(0), Valid=true)
	lpTrade := TradeOutput{
		Order:                        0,
		LedgerClosedAt:               time.Now(),
		SellingAssetType:             "credit_alphanum4",
		BuyingAccountAddress:         "GDEF",
		BuyingAssetType:              "native",
		BuyingOfferID:                null.IntFrom(67890),
		SellingLiquidityPoolID:       null.StringFrom("poolid"),
		LiquidityPoolFee:             null.IntFrom(30),
		HistoryOperationID:           200,
		TradeType:                    2, // liquidity-pool
		RoundingSlippage:             null.IntFrom(0), // MEASURED AS ZERO — Valid=true, Int64=0
		SellerIsExact:                null.BoolFrom(false),
	}

	// Precondition: JSON layer correctly distinguishes the two states
	obJSON, _ := json.Marshal(orderBookTrade.RoundingSlippage)
	lpJSON, _ := json.Marshal(lpTrade.RoundingSlippage)
	if string(obJSON) == string(lpJSON) {
		t.Fatalf("precondition failed: JSON representations should differ, both are %s", string(obJSON))
	}
	t.Logf("JSON correctly distinguishes: order-book=%s, LP=%s", string(obJSON), string(lpJSON))

	// Precondition: the underlying null.Int states are different
	if orderBookTrade.RoundingSlippage.Valid {
		t.Fatal("precondition failed: order-book RoundingSlippage should be invalid (null)")
	}
	if !lpTrade.RoundingSlippage.Valid {
		t.Fatal("precondition failed: LP RoundingSlippage should be valid")
	}

	// Convert both to Parquet
	obParquet := orderBookTrade.ToParquet().(TradeOutputParquet)
	lpParquet := lpTrade.ToParquet().(TradeOutputParquet)

	t.Logf("Parquet RoundingSlippage: order-book=%d, LP=%d", obParquet.RoundingSlippage, lpParquet.RoundingSlippage)

	// THE BUG: Both Parquet rows have identical rounding_slippage = 0,
	// even though JSON correctly distinguishes null vs explicit 0.
	if obParquet.RoundingSlippage == lpParquet.RoundingSlippage {
		t.Errorf("NULL COLLAPSE CONFIRMED: order-book trade (null RoundingSlippage) and LP trade "+
			"(explicit 0 RoundingSlippage) both produce Parquet rounding_slippage=%d — "+
			"the null/zero distinction is destroyed", obParquet.RoundingSlippage)
	}
}
```

### Test Output

```
=== RUN   TestParquetCollapsesNullRoundingSlippageToZero
    data_integrity_poc_test.go:54: JSON correctly distinguishes: order-book=null, LP=0
    data_integrity_poc_test.go:68: Parquet RoundingSlippage: order-book=0, LP=0
    data_integrity_poc_test.go:73: NULL COLLAPSE CONFIRMED: order-book trade (null RoundingSlippage) and LP trade (explicit 0 RoundingSlippage) both produce Parquet rounding_slippage=0 — the null/zero distinction is destroyed
--- FAIL: TestParquetCollapsesNullRoundingSlippageToZero (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.749s
```
