# H003: Trade Parquet export drops `selling_liquidity_pool_id_strkey`

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Liquidity-pool trades should export both the hex liquidity-pool ID and its strkey form on every output surface. For a trade produced from a pool claim atom, the Parquet row should therefore preserve `selling_liquidity_pool_id_strkey` exactly as the JSON row does.

## Mechanism

`TransformTrade()` fills `SellingLiquidityPoolIDStrkey` for `ClaimAtomTypeLiquidityPool`, and the tests assert non-empty strkeys for those rows. But `TradeOutputParquet` omits the field and `TradeOutput.ToParquet()` never copies it, so `export_trades --write-parquet` silently drops the canonical strkey identifier from every liquidity-pool trade.

## Trigger

Run `export_trades --write-parquet` on a ledger containing a liquidity-pool trade (`trade_type = 2`). The JSON output will include `selling_liquidity_pool_id_strkey`, while the Parquet schema/file will omit that column entirely.

## Target Code

- `internal/transform/trade.go:80-157` — populates `SellingLiquidityPoolIDStrkey` for liquidity-pool trades
- `internal/transform/schema.go:285-312` — declares `selling_liquidity_pool_id_strkey` on `TradeOutput`
- `internal/transform/schema_parquet.go:218-244` — defines `TradeOutputParquet` without the strkey field
- `internal/transform/parquet_converter.go:248-274` — writes trade Parquet rows without mapping the strkey
- `cmd/export_trades.go:55-70` — persists those converted rows when `--write-parquet` is enabled

## Evidence

`trade_test.go` already expects concrete non-empty `SellingLiquidityPoolIDStrkey` values such as `LACAKBQ...` and `LAAQEAY...`, proving the transform intentionally exposes the field. The Parquet schema stops at `seller_is_exact`, so the omission is structural rather than data-dependent.

## Anti-Evidence

Orderbook-style trades that come from offers rather than liquidity pools leave the field null anyway, so only `trade_type = 2` rows expose the corruption. JSON exports remain correct.
