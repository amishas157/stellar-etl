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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformTrade()` in `trade.go` computes a strkey-encoded liquidity pool ID via `strkey.Encode()` at line 87 and stores it in the `SellingLiquidityPoolIDStrkey` field of `TradeOutput` at line 156. The JSON schema (`schema.go:311`) declares this field with tag `selling_liquidity_pool_id_strkey`. However, `TradeOutputParquet` in `schema_parquet.go:216-244` has no corresponding field — the struct ends at `SellerIsExact`. The `ToParquet()` converter at `parquet_converter.go:248-274` therefore has no target field to assign the strkey value, silently dropping it. This is the same class of bug as the confirmed success/002 finding (contract event parquet drops operation_id).

### Code Paths Examined

- `internal/transform/trade.go:80-98` — Liquidity pool branch computes both hex (`PoolIDToString`) and strkey (`strkey.Encode`) forms; strkey stored in `liquidityPoolIDStrkey` local variable
- `internal/transform/trade.go:150,156` — `TradeOutput` struct literal assigns both `SellingLiquidityPoolID: liquidityPoolID` and `SellingLiquidityPoolIDStrkey: liquidityPoolIDStrkey`
- `internal/transform/schema.go:305,311` — JSON schema declares both `SellingLiquidityPoolID null.String` and `SellingLiquidityPoolIDStrkey null.String`
- `internal/transform/schema_parquet.go:238,243` — Parquet schema has `SellingLiquidityPoolID string` but NO `SellingLiquidityPoolIDStrkey` field
- `internal/transform/parquet_converter.go:268` — Converter copies `SellingLiquidityPoolID: to.SellingLiquidityPoolID.String` but has no line for the strkey field
- `internal/transform/trade_test.go:789,815` — Test fixtures confirm strkey is populated with values like `LACAKBQ...` and `LAAQEAY...`

### Findings

The bug is confirmed. The `TradeOutputParquet` struct is missing the `SellingLiquidityPoolIDStrkey` field entirely. Since the Parquet struct has no target column, the `ToParquet()` method cannot copy the value, and it is silently dropped. This affects every liquidity-pool trade exported via `--write-parquet`. The hex pool ID survives, so pool identity is not completely lost, but the canonical strkey form (`L...` prefix) that the transform intentionally computes is discarded, breaking format parity between JSON and Parquet outputs.

### PoC Guidance

- **Test file**: `internal/transform/trade_test.go` (or a new `internal/transform/data_integrity_poc_test.go`)
- **Setup**: Use an existing liquidity-pool trade test fixture (the `strictReceiveLPTrade` or `strictSendLPTrade` fixtures from `trade_test.go`)
- **Steps**: Call `TransformTrade()` with a liquidity-pool trade input, then call `ToParquet()` on the resulting `TradeOutput`
- **Assertion**: Assert that the `TradeOutputParquet` struct has a `SellingLiquidityPoolIDStrkey` field matching the JSON output's `SellingLiquidityPoolIDStrkey.String` value. Currently, the Parquet struct has no such field, so the test should fail at compile time or show the value is missing/zeroed.
