# 016: Trade Parquet collapses nullable routing fields into zero values

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformTrade()` preserves nullable routing metadata in `TradeOutput`, including `seller_is_exact` for offer-book trades and `selling_offer_id` for liquidity-pool trades. The Parquet path rewrites those absent states into primitive zero values, and the strongest case is unrecoverable: a manage-offer trade with `seller_is_exact = null` and a strict-send trade with `seller_is_exact = false` both export as the same `seller_is_exact=false` Parquet row.

This is silent structural corruption in normal `export_trades --write-parquet` flow. Some LP-only fields can be partially interpreted with `trade_type`, but `seller_is_exact` cannot be recovered because both colliding rows are ordinary offer trades (`trade_type = 1`).

## Root Cause

`extractClaimedOffers()` leaves `sellerIsExact` unset for manage-offer operations and explicitly sets it for strict-send and strict-receive path payments. `TransformTrade()` carries that nullable state into `TradeOutput`, but `TradeOutputParquet` narrows the affected fields to required primitives and `TradeOutput.ToParquet()` serializes raw `.Bool`, `.Int64`, and `.String` members without checking `Valid`, turning null into `false`, `0`, or `""`.

## Reproduction

Run `TransformTrade()` on the existing trade fixtures for a manage-sell-offer operation, a path-payment-strict-send operation, and a liquidity-pool trade. At the JSON/output-struct layer, the manage-offer trade has `SellerIsExact.Valid == false`, the strict-send trade has `SellerIsExact.Valid == true` with `Bool == false`, and the LP trade has `SellingOfferID.Valid == false`. Convert them through `ToParquet()`: the first two rows both export `seller_is_exact=false`, and the LP row exports `selling_offer_id=0`.

## Affected Code

- `internal/transform/trade.go:80-114` — assigns offer-only versus liquidity-pool-only trade fields.
- `internal/transform/trade.go:164-254` — leaves `sellerIsExact` null for manage-offer trades and sets it only for strict-send/strict-receive path payments.
- `internal/transform/schema.go:286-311` — models the affected trade fields as nullable `null.Int`, `null.String`, and `null.Bool`.
- `internal/transform/schema_parquet.go:219-244` — narrows the same fields to required Parquet primitives.
- `internal/transform/parquet_converter.go:248-274` — serializes `.Int64`, `.String`, and `.Bool` directly, discarding nullability.
- `cmd/export_trades.go:55-70` — uses this Parquet conversion path in the live exporter.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTradeParquetNullCollapse`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go test ./internal/transform/... -run TestTradeParquetNullCollapse -v`.

### Test Body

```go
package transform

import "testing"

func TestTradeParquetNullCollapse(t *testing.T) {
	t.Run("seller_is_exact null and false collide for offer trades", func(t *testing.T) {
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

		manageBuyTrade := manageOfferTrades[0]
		strictSendTrade := strictSendTrades[0]
		manageBuyParquet := manageBuyTrade.ToParquet().(TradeOutputParquet)
		strictSendParquet := strictSendTrade.ToParquet().(TradeOutputParquet)

		if manageBuyTrade.SellerIsExact.Valid {
			t.Fatal("expected manage-buy trade seller_is_exact to be absent in source output")
		}
		if !strictSendTrade.SellerIsExact.Valid || strictSendTrade.SellerIsExact.Bool {
			t.Fatal("expected strict-send trade seller_is_exact to be explicitly false in source output")
		}
		if manageBuyParquet.SellerIsExact != strictSendParquet.SellerIsExact {
			t.Fatalf("expected parquet outputs to collide, got %v and %v", manageBuyParquet.SellerIsExact, strictSendParquet.SellerIsExact)
		}
		if manageBuyParquet.SellerIsExact != false {
			t.Fatalf("expected collided parquet value to be false, got %v", manageBuyParquet.SellerIsExact)
		}
	})

	t.Run("liquidity-pool trade loses absent selling_offer_id", func(t *testing.T) {
		transaction := makeTradeTestInput()

		lpTrades, err := TransformTrade(4, 100, transaction, genericCloseTime)
		if err != nil {
			t.Fatalf("TransformTrade liquidity-pool trade failed: %v", err)
		}
		if len(lpTrades) != 1 {
			t.Fatalf("expected 1 liquidity-pool trade, got %d", len(lpTrades))
		}

		lpTrade := lpTrades[0]
		lpParquet := lpTrade.ToParquet().(TradeOutputParquet)

		if lpTrade.SellingOfferID.Valid {
			t.Fatal("expected selling_offer_id to be absent in source output for liquidity-pool trade")
		}
		if lpParquet.SellingOfferID != 0 {
			t.Fatalf("expected absent selling_offer_id to collapse to 0 in parquet, got %d", lpParquet.SellingOfferID)
		}
	})
}
```

## Expected vs Actual Behavior

- **Expected**: Absent trade-routing fields should remain absent in Parquet, so manage-offer `seller_is_exact` stays null, strict-send `seller_is_exact` stays explicit `false`, and liquidity-pool trades keep `selling_offer_id` null.
- **Actual**: The Parquet export rewrites absent values to required zero values such as `seller_is_exact=false` and `selling_offer_id=0`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses the real `TransformTrade()` fixture path and the production `TradeOutput.ToParquet()` converter.
2. Realistic preconditions: YES — manage-offer, strict-send, and liquidity-pool trades are standard Stellar operations already exercised by this package's fixtures.
3. Bug vs by-design: BUG — the JSON/export schema already models these fields as nullable, and the Parquet path silently changes their meaning without preserving an alternate null sentinel.
4. Final severity: High — this is silent structural corruption of non-financial trade-routing data, with an unrecoverable collision inside `trade_type = 1`.
5. In scope: YES — it is a concrete export-layer data corruption bug in normal operation.
6. Test correctness: CORRECT — the test proves the source states differ before conversion and only collide after the Parquet conversion step.
7. Alternative explanations: NONE — `trade_type` only partially explains some LP-only fields and cannot recover the `seller_is_exact` collision.
8. Novelty: NOT ASSESSED — duplicate handling is performed by the orchestrator.

## Suggested Fix

Make the affected Parquet fields nullable, such as optional pointer fields with appropriate Parquet tags, and emit `nil` when the source `null.*` wrapper is invalid. At minimum, fix `seller_is_exact`, `selling_offer_id`, `selling_liquidity_pool_id`, `liquidity_pool_fee`, and `rounding_slippage`, then audit the rest of `parquet_converter.go` for similar nullable-to-required collapses.
