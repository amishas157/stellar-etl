# H003: Trade parquet export collapses nullable trade metadata to zero values

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_trades --write-parquet` should preserve the same null semantics as `TradeOutput`: fields that are not applicable to a trade row should stay null/absent, and liquidity-pool rows should retain their `selling_liquidity_pool_id_strkey`. For a classic orderbook trade like the repository's `offerOneOutput`, parquet should not invent `selling_liquidity_pool_id=""`, `liquidity_pool_fee=0`, `rounding_slippage=0`, or `seller_is_exact=false`.

## Mechanism

`TradeOutput` intentionally uses `null.Int`, `null.String`, and `null.Bool` to distinguish classic orderbook trades from liquidity-pool and path-payment-specific metadata. But `TradeOutputParquet` replaces those nullable fields with plain `int64`, `string`, and `bool`, drops `SellingLiquidityPoolIDStrkey` completely, and `TradeOutput.ToParquet()` writes `.Int64`, `.String`, and `.Bool` zero values. That means parquet consumers cannot tell “not applicable” from real zero/false values, and liquidity-pool rows lose the strkey ID entirely.

## Trigger

Run `export_trades --write-parquet` on any ledger range containing:
1. a normal orderbook trade, or
2. a liquidity-pool trade.

For orderbook trades, JSON has null pool/slippage/exactness metadata while parquet emits empty/zero/false values. For liquidity-pool trades, JSON has `selling_liquidity_pool_id_strkey`, but parquet has no such column.

## Target Code

- `internal/transform/schema.go:TradeOutput:285-312` — JSON schema models several trade fields as nullable and includes `SellingLiquidityPoolIDStrkey`
- `internal/transform/trade.go:TransformTrade:80-156` — leaves classic-trade pool fields unset and populates LP/path-payment-only fields conditionally
- `internal/transform/trade_test.go:730-815` — existing expected outputs show classic trades with nil pool metadata and LP trades with populated `SellingLiquidityPoolIDStrkey`, `LiquidityPoolFee`, `RoundingSlippage`, and `SellerIsExact`
- `internal/transform/schema_parquet.go:TradeOutputParquet:218-244` — parquet schema removes nullability and omits `selling_liquidity_pool_id_strkey`
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:248-274` — converter serializes null wrappers as primitive zero values
- `cmd/export_trades.go:tradesCmd.Run:47-70` — command writes parquet rows from the lossy schema

## Evidence

The trade transformer and tests already encode “applicable vs not applicable” as part of the output contract. The parquet path discards that distinction for exactly the fields that differentiate orderbook trades, liquidity-pool trades, and path-payment slippage metadata.

## Anti-Evidence

Some downstream parquet readers may conventionally treat `0` or `false` as sentinel values. But this repository's own JSON schema and tests say those fields are nullable, so converting null into a primitive value is a semantic change, not a harmless encoding choice.
