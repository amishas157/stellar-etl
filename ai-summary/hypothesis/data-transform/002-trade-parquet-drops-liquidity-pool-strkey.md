# H002: Trade Parquet drops populated `selling_liquidity_pool_id_strkey`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a trade comes from a liquidity pool, the Parquet row should preserve the same `selling_liquidity_pool_id_strkey` value that the JSON row exports. Downstream analytics should not lose the canonical Stellar `L...` pool identifier just because the file format changes.

## Mechanism

`TransformTrade()` computes both the hex `selling_liquidity_pool_id` and the strkey `selling_liquidity_pool_id_strkey` for liquidity-pool claim atoms, and `TradeOutput` keeps both fields. But `TradeOutputParquet` has no `SellingLiquidityPoolIDStrkey` column, so `TradeOutput.ToParquet()` has nowhere to copy the populated value and every Parquet export silently drops it.

## Trigger

Export trades with `--write-parquet` for any ledger containing a liquidity-pool trade. The JSON row will contain a non-empty `selling_liquidity_pool_id_strkey` such as `LACAK...`, while the Parquet schema will have no corresponding column at all.

## Target Code

- `internal/transform/trade.go:80-98` — liquidity-pool trades compute both hex and strkey pool identifiers
- `internal/transform/trade.go:148-156` — `TradeOutput` stores `SellingLiquidityPoolIDStrkey`
- `internal/transform/schema.go:305-311` — JSON schema exposes `selling_liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:236-244` — Parquet schema includes `selling_liquidity_pool_id` but omits the strkey companion field
- `internal/transform/parquet_converter.go:266-274` — converter copies the hex pool ID but never copies a strkey value
- `internal/transform/trade_test.go:766-816` — liquidity-pool fixtures already populate `SellingLiquidityPoolIDStrkey`
- `cmd/export_trades.go:55-70` — live parquet export path writes `TradeOutputParquet`

## Evidence

The liquidity-pool fixtures in `trade_test.go` expect non-empty `SellingLiquidityPoolIDStrkey` values for both strict-send and strict-receive pool trades. The Parquet schema immediately below `SellingLiquidityPoolID` stops at `SellerIsExact`, and the converter struct literal has no field for the strkey identifier, so the populated JSON value is unrepresentable in Parquet.

## Anti-Evidence

The hex `selling_liquidity_pool_id` still survives, so the raw pool identity is not totally lost. But the transform intentionally exports the strkey form as a separate column, and dropping it from Parquet breaks format consistency for consumers that join or display the canonical `L...` identifier.
