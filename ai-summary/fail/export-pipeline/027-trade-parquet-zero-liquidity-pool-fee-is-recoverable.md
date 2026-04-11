# H027: Trade parquet `liquidity_pool_fee = 0` for orderbook rows is recoverable from `trade_type`

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Orderbook trades should preserve `liquidity_pool_fee = NULL`, matching the JSON export and indicating that no pool fee applies. Only liquidity-pool trades should emit a populated fee.

## Mechanism

`TradeOutput` keeps `liquidity_pool_fee` as `null.Int`, but the parquet schema requires `int64` and `ToParquet()` writes `to.LiquidityPoolFee.Int64` without checking `Valid`. That makes non-pool trades export `0`, which initially looks like a fabricated fee value.

## Trigger

Export an orderbook trade through `export_trades --write-parquet`. The JSON row will have no `liquidity_pool_fee`, while the parquet row will contain `0`.

## Target Code

- `internal/transform/trade.go:80-114` — orderbook trades leave `outputPoolFee` unset and mark `trade_type = 1`; liquidity-pool trades populate both.
- `internal/transform/schema.go:306-309` — JSON schema models `liquidity_pool_fee` as nullable.
- `internal/transform/schema_parquet.go:239-243` — parquet schema makes `liquidity_pool_fee` required.
- `internal/transform/parquet_converter.go:268-273` — converter writes `.Int64` unconditionally.

## Evidence

The transform has two distinct states at the JSON layer: null fee for orderbook trades and populated fee for liquidity-pool trades. The parquet converter discards that validity bit and therefore exports zero for every orderbook row.

## Anti-Evidence

`trade_type` is exported on the same row and already distinguishes orderbook (`1`) from liquidity-pool (`2`) trades. Because the applicability of the fee is recoverable from that discriminator, the zero value does not create an unrecoverable ambiguity.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The parquet row still contains enough information to reconstruct the intended semantics: `trade_type = 1` means the fee is inapplicable, not literally zero. That puts this in the same recoverable-null-collapse category as other rejected trade-parquet candidates.

### Lesson Learned

Null-to-zero on its own is not enough. The key question is whether the exported row loses information or merely requires a deterministic post-processing rule from another column already present.
