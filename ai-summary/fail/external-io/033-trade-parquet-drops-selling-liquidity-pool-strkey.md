# H002: Trade Parquet drops `selling_liquidity_pool_id_strkey`

**Date**: 2026-04-15
**Subsystem**: external-io
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For liquidity-pool-backed trade rows, `export_trades --write-parquet` should preserve the same populated `selling_liquidity_pool_id_strkey` that JSON exports expose. A Parquet consumer should be able to recover both the hex pool ID and the canonical `L...` StrKey for the selling liquidity pool.

## Mechanism

`TransformTrade()` computes both `SellingLiquidityPoolID` and `SellingLiquidityPoolIDStrkey` for `ClaimAtomTypeLiquidityPool` trades, but `TradeOutputParquet` has no `SellingLiquidityPoolIDStrkey` column and `TradeOutput.ToParquet()` never copies it. The Parquet export therefore silently strips the canonical pool identifier from every liquidity-pool trade row while leaving the rest of the row structurally valid.

## Trigger

Run `export_trades --write-parquet` on any ledger range containing a liquidity-pool trade, such as a path-payment or pool trade that claims from `ClaimAtomTypeLiquidityPool`. The JSON row will contain both `selling_liquidity_pool_id` and `selling_liquidity_pool_id_strkey`, but the Parquet row can only retain the hex-form ID.

## Target Code

- `internal/transform/trade.go:TransformTrade:80-92,131-157` — derives `liquidityPoolIDStrkey` and stores it on `TradeOutput`
- `internal/transform/schema.go:TradeOutput:285-312` — JSON/output schema includes `selling_liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:TradeOutputParquet:218-244` — Parquet schema omits that field
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:248-274` — converter writes `SellingLiquidityPoolID` but has no StrKey mapping
- `cmd/export_trades.go:tradesCmd.Run:55-70` — production Parquet path writes `TradeOutputParquet`

## Evidence

The trade transformer explicitly builds the StrKey with `strkey.Encode(strkey.VersionByteLiquidityPool, id[:])` and assigns it to `SellingLiquidityPoolIDStrkey` only for the liquidity-pool branch, so this is not a hypothetical convenience field. The loss happens later, because the Parquet row type simply lacks any field that could preserve the value.

## Anti-Evidence

Only liquidity-pool trade rows are affected; orderbook trades legitimately leave the field null. The surviving hex `selling_liquidity_pool_id` still identifies the pool, but downstream systems expecting JSON/Parquet parity lose the canonical encoded identifier without any error.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-transform/005-trade-parquet-drops-liquidity-pool-strkey.md.gh-published` and `ai-summary/success/cli-commands/005-trade-parquet-omits-liquidity-pool-strkey.md.gh-published`
**Failed At**: reviewer

### Trace Summary

The hypothesis correctly identifies that `TradeOutputParquet` (schema_parquet.go:218-244) lacks a `SellingLiquidityPoolIDStrkey` field, while `TradeOutput` (schema.go:311) has one, and `ToParquet()` (parquet_converter.go:248-274) therefore cannot carry the value. The bug is real and all code paths are confirmed. However, this exact finding has already been confirmed and published twice under both the `data-transform` and `cli-commands` subsystems.

### Code Paths Examined

- `internal/transform/schema.go:TradeOutput:311` — confirmed `SellingLiquidityPoolIDStrkey null.String` field present
- `internal/transform/schema_parquet.go:TradeOutputParquet:218-244` — confirmed no `SellingLiquidityPoolIDStrkey` field
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:248-274` — confirmed no `SellingLiquidityPoolIDStrkey` assignment

### Why It Failed

This is an exact duplicate of two already-confirmed findings:
1. `ai-summary/success/data-transform/005-trade-parquet-drops-liquidity-pool-strkey.md.gh-published`
2. `ai-summary/success/cli-commands/005-trade-parquet-omits-liquidity-pool-strkey.md.gh-published`

All three identify the same root cause (missing `SellingLiquidityPoolIDStrkey` in `TradeOutputParquet` and `ToParquet()`), same affected code paths, and same impact. The only difference is the subsystem label.

### Lesson Learned

The "trade Parquet drops liquidity-pool strkey" finding is already well-documented across multiple subsystems. Always check success directories across ALL subsystems before filing, not just the target subsystem.
