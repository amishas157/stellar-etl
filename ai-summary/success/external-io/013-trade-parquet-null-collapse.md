# 013: Trade Parquet null collapse

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

Trade export models several routing fields as nullable because they are only meaningful for specific trade sources, but the Parquet conversion rewrites those nullable wrappers to plain scalars. That silently collapses absent values onto real zero/false values. The unrecoverable case is `seller_is_exact`: an orderbook manage-offer trade and an orderbook strict-send path-payment trade both export `trade_type=1`, but Parquet rewrites the former's absent value to the latter's concrete `false`.

## Root Cause

`TransformTrade()` and `extractClaimedOffers()` intentionally leave `SellerIsExact`, `RoundingSlippage`, `LiquidityPoolFee`, and related routing fields unset when they do not apply. `TradeOutput.ToParquet()` then reads `.Int64`, `.String`, and `.Bool` from `guregu/null` wrappers and stores them into a Parquet schema that has no nullable representation for those columns, erasing the distinction between null and concrete zero values.

## Reproduction

During normal export, manage-buy/manage-sell offer trades leave `seller_is_exact` unset, while `PathPaymentStrictSend` sets it to `false`. Both still emit `trade_type=1` orderbook rows. JSON preserves `null` versus `false`, but the Parquet export rewrites both rows to `seller_is_exact=false`, so downstream consumers cannot tell whether `false` was explicit or synthesized from absence.

## Affected Code

- `internal/transform/trade.go:TransformTrade:80-156` — populates nullable trade metadata only when the trade source provides it.
- `internal/transform/trade.go:extractClaimedOffers:164-254` — leaves `sellerIsExact` null for manage-offer trades and sets it to `false` for strict-send path payments.
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:248-274` — flattens nullable trade fields by reading `.Int64`, `.String`, and `.Bool`.
- `internal/transform/schema_parquet.go:TradeOutputParquet:220-244` — defines the affected Parquet columns as plain scalars with no nullability marker.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTradeNullFieldsCollapseInParquet`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/guregu/null"
)

func TestTradeNullFieldsCollapseInParquet(t *testing.T) {
	manageBuyTrade := TradeOutput{
		TradeType:        1,
		SellingOfferID:   null.IntFrom(111),
		BuyingOfferID:    null.IntFrom(222),
		SellerIsExact:    null.Bool{},
		RoundingSlippage: null.Int{},
		LiquidityPoolFee: null.Int{},
	}

	strictSendTrade := manageBuyTrade
	strictSendTrade.SellerIsExact = null.BoolFrom(false)

	manageBuyJSON, err := json.Marshal(manageBuyTrade)
	if err != nil {
		t.Fatalf("marshal manage-buy trade: %v", err)
	}
	strictSendJSON, err := json.Marshal(strictSendTrade)
	if err != nil {
		t.Fatalf("marshal strict-send trade: %v", err)
	}

	var manageBuyRow map[string]any
	if err := json.Unmarshal(manageBuyJSON, &manageBuyRow); err != nil {
		t.Fatalf("unmarshal manage-buy json: %v", err)
	}
	var strictSendRow map[string]any
	if err := json.Unmarshal(strictSendJSON, &strictSendRow); err != nil {
		t.Fatalf("unmarshal strict-send json: %v", err)
	}

	if manageBuyRow["seller_is_exact"] != nil {
		t.Fatalf("manage-buy json should preserve seller_is_exact=null, got %#v", manageBuyRow["seller_is_exact"])
	}
	if strictSendRow["seller_is_exact"] != false {
		t.Fatalf("strict-send json should preserve seller_is_exact=false, got %#v", strictSendRow["seller_is_exact"])
	}

	manageBuyParquet := manageBuyTrade.ToParquet().(TradeOutputParquet)
	strictSendParquet := strictSendTrade.ToParquet().(TradeOutputParquet)

	if manageBuyParquet.TradeType != 1 || strictSendParquet.TradeType != 1 {
		t.Fatalf("expected both rows to remain orderbook trades, got %d and %d", manageBuyParquet.TradeType, strictSendParquet.TradeType)
	}
	if manageBuyParquet.SellerIsExact != strictSendParquet.SellerIsExact {
		t.Fatalf("expected parquet values to collide, got %v and %v", manageBuyParquet.SellerIsExact, strictSendParquet.SellerIsExact)
	}
	if manageBuyParquet.SellerIsExact != false {
		t.Fatalf("expected null seller_is_exact to flatten to false, got %v", manageBuyParquet.SellerIsExact)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Parquet should preserve the distinction between absent trade-routing metadata and explicit zero/false values, either via nullable columns or separate validity fields.
- **Actual**: Parquet rewrites absent trade-routing metadata to scalar defaults; `seller_is_exact=null` becomes `seller_is_exact=false`, creating collisions with real strict-send orderbook rows.

## Adversarial Review

1. Exercises claimed bug: YES — the test hits `TradeOutput.ToParquet()`, the production conversion layer that erases nullability.
2. Realistic preconditions: YES — `extractClaimedOffers()` produces null `seller_is_exact` for manage-offer trades and explicit `false` for strict-send path payments, and both export `trade_type=1`.
3. Bug vs by-design: BUG — keeping the column while silently changing null semantics is materially different from intentionally omitting a JSON-only convenience field.
4. Final severity: High — the export emits structurally wrong Parquet data that downstream analytics can consume without error.
5. In scope: YES — this is silent data corruption in a shipped export format.
6. Test correctness: CORRECT — the rewritten PoC uses production data types, checks JSON versus Parquet semantics directly, and avoids unreachable field combinations from the original draft.
7. Alternative explanations: NONE
8. Novelty: NOT ASSESSED — overlapping confirmed summaries already exist, and duplicate handling is delegated to the orchestrator.

## Suggested Fix

Make the affected Parquet columns nullable, or add parallel validity columns so absent values remain distinguishable from explicit zero/false values. At minimum, `seller_is_exact` needs a representation that preserves null versus `false` for `trade_type=1` rows.
