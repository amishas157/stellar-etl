# H004: Trade parquet export omits `selling_liquidity_pool_id_strkey` for pool trades

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_trades --write-parquet` exports liquidity-pool trades, parquet rows should preserve the same `selling_liquidity_pool_id_strkey` that the JSON rows expose for those trades. Downstream systems should be able to join pool trades on the canonical strkey pool identifier without having to fall back to a different export format.

## Mechanism

`TransformTrade` computes both `selling_liquidity_pool_id` and `selling_liquidity_pool_id_strkey` when the claim atom is a liquidity-pool trade. The JSON schema includes the strkey field and the tests assert it. But `TradeOutputParquet` and `TradeOutput.ToParquet()` omit that field entirely, so parquet silently drops the canonical pool identifier for every liquidity-pool trade while JSON from the same run keeps it.

## Trigger

Run `export_trades` with `--write-parquet` over a range containing liquidity-pool trades. Compare JSON and parquet output for those rows: JSON includes `selling_liquidity_pool_id_strkey`, while parquet has no column for it.

## Target Code

- `internal/transform/trade.go:80-98` — liquidity-pool trades compute both raw and strkey pool IDs
- `internal/transform/trade.go:131-157` — transformed trade rows populate `SellingLiquidityPoolIDStrkey`
- `internal/transform/schema.go:285-312` — JSON trade schema includes `selling_liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:218-244` — parquet trade schema omits the strkey field
- `internal/transform/parquet_converter.go:248-275` — trade parquet conversion has no assignment for the strkey field
- `cmd/export_trades.go:46-70` — command writes the same transformed trades to JSON and parquet

## Evidence

The trade transform sets `SellingLiquidityPoolIDStrkey: liquidityPoolIDStrkey`, and `internal/transform/trade_test.go:789-815` asserts non-empty values for pool trades. The parquet schema ends with `seller_is_exact`, so the strkey never reaches parquet output.

## Anti-Evidence

Classic orderbook trades do not populate a liquidity-pool identifier in either format. The issue only affects liquidity-pool trade rows, but those are exactly the rows where the missing identifier matters.
