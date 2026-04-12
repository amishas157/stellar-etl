# H021: Trade Parquet drops `selling_liquidity_pool_id_strkey`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For liquidity-pool trades, Parquet should preserve the same canonical `L...` liquidity-pool identifier that the JSON trade row exposes in `selling_liquidity_pool_id_strkey`. Exporting Parquet should not silently erase one of the trade identifiers that the transform already computed.

## Mechanism

`TransformTrade()` derives both the hex pool ID and the canonical StrKey form for liquidity-pool claim atoms, then stores both on `TradeOutput`. The Parquet schema and converter only carry `selling_liquidity_pool_id`, never `selling_liquidity_pool_id_strkey`, so every liquidity-pool trade row loses its canonical `L...` identifier when `export_trades --write-parquet` is used.

## Trigger

Run `export_trades --write-parquet` on any range containing a liquidity-pool trade (`trade_type=2`). The JSON row will contain both `selling_liquidity_pool_id` and `selling_liquidity_pool_id_strkey`, while the Parquet row will only retain the hex ID.

## Target Code

- `internal/transform/trade.go:TransformTrade:85-92` — encodes the liquidity-pool ID into StrKey form
- `internal/transform/trade.go:TransformTrade:131-156` — assigns `SellingLiquidityPoolIDStrkey` onto the JSON/output struct
- `internal/transform/schema.go:TradeOutput:303-311` — exposes `selling_liquidity_pool_id_strkey` in the export schema
- `internal/transform/schema_parquet.go:TradeOutputParquet:218-244` — defines no Parquet column for that field
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:248-274` — never copies the StrKey field into the Parquet row
- `cmd/export_trades.go:tradesCmd.Run:68-70` — writes those Parquet rows during production exports

## Evidence

The trade transform computes the StrKey value with `strkey.Encode(...)` specifically for liquidity-pool claim atoms and stores it alongside the hex ID. `internal/transform/trade_test.go:783-815` contains expected liquidity-pool trade outputs with populated `SellingLiquidityPoolIDStrkey`, confirming that the field is part of the intended exported row shape. The Parquet schema has no matching column.

## Anti-Evidence

The raw hex `selling_liquidity_pool_id` still survives in Parquet, so downstream systems could theoretically re-encode it themselves. But that requires extra decoding logic that JSON users do not need, and the Parquet artifact no longer matches the exporter’s own row schema.
