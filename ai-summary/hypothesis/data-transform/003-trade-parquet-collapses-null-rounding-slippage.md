# H003: Trade Parquet collapses null `rounding_slippage` into explicit `0`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Parquet trade rows should preserve the same three `rounding_slippage` states that the JSON transform can emit: null when the concept does not apply, `0` when a liquidity-pool trade had no rounding loss, and a positive or sentinel value when rounding slippage was measured. Order-book trades should not be exported as though they had exactly zero rounding slippage.

## Mechanism

`TradeOutput` models `RoundingSlippage` as `null.Int`, and the fixtures already show both null and explicit `0` as valid JSON-layer states. `TradeOutputParquet` narrows the field to required `int64`, and `TradeOutput.ToParquet()` writes `to.RoundingSlippage.Int64`, so null order-book trades and exact-zero liquidity-pool trades both serialize as `rounding_slippage = 0`.

## Trigger

Run `export_trades --write-parquet` on a ledger range containing both an order-book trade and a liquidity-pool trade whose computed `rounding_slippage` is exactly zero. The JSON rows remain distinguishable (`null` versus `0`), but the Parquet rows export the same numeric value `0` for both cases.

## Target Code

- `internal/transform/schema.go:TradeOutput:303-311` — JSON/output schema models `rounding_slippage` as `null.Int`
- `internal/transform/trade.go:80-105` — only liquidity-pool trades can populate `roundingSlippageBips`; order-book trades leave it null
- `internal/transform/trade.go:148-156` — transformed trades carry the nullable `RoundingSlippage` field into output rows
- `internal/transform/trade.go:350-399` — `roundingSlippage()` returns explicit numeric values, including real `0`
- `internal/transform/schema_parquet.go:TradeOutputParquet:236-243` — Parquet schema narrows `rounding_slippage` to required `int64`
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:266-273` — converter serializes `to.RoundingSlippage.Int64` without checking validity
- `internal/transform/trade_test.go:738-742,782-789` — fixtures show order-book trades with null slippage and liquidity-pool trades with explicit `null.IntFrom(0)`

## Evidence

The trade fixtures already establish the collision: the order-book examples at `trade_test.go:738-742` have no `RoundingSlippage` field set, while the liquidity-pool example at `trade_test.go:782-789` sets `RoundingSlippage: null.IntFrom(0)`. Because the Parquet converter writes `.Int64` into a required `int64` column, both rows become indistinguishable `0` in Parquet.

## Anti-Evidence

Downstream consumers can partially reconstruct applicability from `trade_type`, because the null state appears on order-book trades while explicit values occur on liquidity-pool trades. But the field itself is still corrupted: Parquet claims an exact numeric slippage value where the JSON export correctly says the metric was not applicable at all.
