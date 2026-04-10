# 004: Trade Parquet flattens nullable `seller_is_exact` to `false`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformTrade()` preserves three distinct `seller_is_exact` states at the JSON layer: `null` for manage-offer trades where the concept does not apply, `false` for path-payment-strict-send trades, and `true` for path-payment-strict-receive trades. The Parquet path collapses the first two by converting `null.Bool{}` and `null.BoolFrom(false)` into the same required `bool` value `false`.

This is real structural corruption in normal export flow. Existing trade fixtures already exercise both manage-offer and strict-send paths, and both emit `trade_type=1`, so Parquet consumers cannot recover the lost semantic distinction from the trade row itself.

## Root Cause

`extractClaimedOffers()` leaves `sellerIsExact` unset for manage-offer operations and explicitly sets it for strict-send/strict-receive path payments. `TradeOutput` correctly models that as `null.Bool`, but `TradeOutputParquet` narrows the field to a required `bool` and `TradeOutput.ToParquet()` writes `to.SellerIsExact.Bool` without checking `Valid`, turning null into false.

## Reproduction

Use the existing trade test fixtures to transform one manage-sell-offer operation and one path-payment-strict-send operation. At the JSON layer, the manage-offer trade has `SellerIsExact.Valid == false` while the strict-send trade has `SellerIsExact.Valid == true` and `Bool == false`. Convert both through `ToParquet()`: each row exports `seller_is_exact=false`, proving the null state was silently lost.

## Affected Code

- `internal/transform/trade.go:TransformTrade:31-37,131-156` — propagates `sellerIsExact` from `extractClaimedOffers()` into `TradeOutput`.
- `internal/transform/trade.go:extractClaimedOffers:180-254` — leaves manage-offer rows null and explicitly sets strict-send/strict-receive rows to false/true.
- `internal/transform/schema.go:289-311` — models `seller_is_exact` as `null.Bool` in the JSON/output schema.
- `internal/transform/schema_parquet.go:224-244` — declares `seller_is_exact` as a required Parquet `bool`.
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:248-274` — serializes `to.SellerIsExact.Bool`, ignoring nullability.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTradeParquetCollapsesNullSellerIsExact`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go test ./internal/transform/... -run TestTradeParquetCollapsesNullSellerIsExact -v`.

### Test Body

```go
package transform

import "testing"

func TestTradeParquetCollapsesNullSellerIsExact(t *testing.T) {
	transaction := makeTradeTestInput()

	manageOfferTrades, err := TransformTrade(0, 100, transaction, genericCloseTime)
	if err != nil {
		t.Fatalf("TransformTrade manage-offer failed: %v", err)
	}
	if len(manageOfferTrades) != 1 {
		t.Fatalf("expected 1 manage-offer trade, got %d", len(manageOfferTrades))
	}

	strictSendTrades, err := TransformTrade(2, 100, transaction, genericCloseTime)
	if err != nil {
		t.Fatalf("TransformTrade strict-send failed: %v", err)
	}
	if len(strictSendTrades) == 0 {
		t.Fatal("expected at least 1 strict-send trade")
	}

	manageOfferTrade := manageOfferTrades[0]
	strictSendTrade := strictSendTrades[0]

	if manageOfferTrade.SellerIsExact.Valid {
		t.Fatalf("precondition failed: manage-offer trade should leave SellerIsExact null, got %+v", manageOfferTrade.SellerIsExact)
	}
	if !strictSendTrade.SellerIsExact.Valid || strictSendTrade.SellerIsExact.Bool {
		t.Fatalf("precondition failed: strict-send trade should set SellerIsExact=false, got %+v", strictSendTrade.SellerIsExact)
	}

	manageOfferParquet := manageOfferTrade.ToParquet().(TradeOutputParquet)
	strictSendParquet := strictSendTrade.ToParquet().(TradeOutputParquet)

	if manageOfferParquet.SellerIsExact == strictSendParquet.SellerIsExact {
		t.Fatalf("BUG CONFIRMED: parquet collapses null and explicit false seller_is_exact into the same value %v", manageOfferParquet.SellerIsExact)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Manage-offer trades should preserve `seller_is_exact` as null/not-applicable in Parquet, while strict-send path payments should preserve explicit `false`.
- **Actual**: Both states export as the same Parquet boolean `false`.

## Adversarial Review

1. Exercises claimed bug: YES — the test uses the real `TransformTrade()` path on existing trade fixtures for manage-offer and strict-send operations, then calls the production `ToParquet()` converter.
2. Realistic preconditions: YES — both operation types are standard Stellar trade-producing operations already covered by this package's fixture set.
3. Bug vs by-design: BUG — the exporter already chose nullable semantics in `TradeOutput`; only the Parquet schema/converter destroys them.
4. Final severity: High — this is silent structural corruption of a non-financial analytics field.
5. In scope: YES — it is a concrete transform-layer export bug that produces plausible but wrong Parquet data.
6. Test correctness: CORRECT — the PoC proves the source JSON-layer states differ before conversion and fails only because the Parquet values collide.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Make `TradeOutputParquet.SellerIsExact` nullable, such as `*bool` with an optional Parquet field tag, and emit `nil` when `to.SellerIsExact.Valid` is false. Audit similar nullable-to-required conversions in `parquet_converter.go` so other tri-state fields do not silently collapse.
