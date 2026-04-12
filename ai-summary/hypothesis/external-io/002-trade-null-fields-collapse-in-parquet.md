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
