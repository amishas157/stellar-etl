# H026: Trade parquet empty-string `selling_liquidity_pool_id` is recoverable from `trade_type`

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Classic orderbook trades should preserve `selling_liquidity_pool_id = NULL`, matching the JSON export and indicating that no liquidity pool participated on the selling side. Liquidity-pool trades should continue to carry the populated raw pool ID.

## Mechanism

`TradeOutput` models `selling_liquidity_pool_id` as `null.String`, but `TradeOutputParquet` uses a required `string` and `ToParquet()` copies `to.SellingLiquidityPoolID.String` unconditionally. That means orderbook rows serialize as `""` instead of null, which at first glance looks like fabricated pool identity in parquet output.

## Trigger

Export any normal orderbook trade with `--write-parquet`. The JSON row will omit or null out `selling_liquidity_pool_id`, while the parquet row will contain an empty string.

## Target Code

- `internal/transform/schema.go:305-311` — JSON trade schema keeps `selling_liquidity_pool_id` nullable and carries `trade_type`.
- `internal/transform/schema_parquet.go:238-243` — parquet schema makes `selling_liquidity_pool_id` a required string.
- `internal/transform/parquet_converter.go:248-274` — `ToParquet()` copies `to.SellingLiquidityPoolID.String` without checking validity.

## Evidence

The JSON/parquet schemas disagree on nullability, and the converter has no validity guard. Orderbook trades set `trade_type = 1` and leave the pool ID unset in `TransformTrade()`, so parquet necessarily emits the zero string.

## Anti-Evidence

The same parquet row also carries `trade_type`, and `trade_type = 1` unambiguously means "not a liquidity-pool trade." Downstream consumers can therefore recover the intended null semantics from the row itself.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is another discriminator-recoverable null collapse. Unlike confirmed trade bugs where two semantically different states collapsed within the same `trade_type`, the empty-string pool ID is fully explained by `trade_type = 1` on the same row.

### Lesson Learned

For trade parquet nullability candidates, always ask whether `trade_type` (or another sibling discriminator) lets consumers reconstruct the missing null state. If yes, the column is noisy but not meaningfully corrupted.
