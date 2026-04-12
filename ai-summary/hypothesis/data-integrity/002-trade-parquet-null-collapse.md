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
