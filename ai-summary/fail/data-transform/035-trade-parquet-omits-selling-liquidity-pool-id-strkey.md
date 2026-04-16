# H005: Trade Parquet Omits `selling_liquidity_pool_id_strkey`

**Date**: 2026-04-16
**Subsystem**: data-transform
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For liquidity-pool trades, the Parquet row should preserve the canonical strkey pool identifier in `selling_liquidity_pool_id_strkey`, matching the JSON output. Downstream consumers should be able to join LP trade rows against other strkey-based pool datasets without having to re-encode the raw pool ID themselves.

## Mechanism

`TransformTrade()` computes both `selling_liquidity_pool_id` (hex/raw string form) and `selling_liquidity_pool_id_strkey`, and the JSON schema exposes both fields. But `TradeOutputParquet` has no `selling_liquidity_pool_id_strkey` column and `TradeOutput.ToParquet()` never copies it, so LP trade rows exported to Parquet silently lose the canonical pool string while JSON keeps it. This creates a format-dependent discrepancy on a real trade identifier column, not just a cosmetic naming difference.

## Trigger

1. Run `export_trades --write-parquet` on any ledger range containing LP trades.
2. Compare a JSON LP-trade row to the corresponding Parquet row.
3. JSON contains `selling_liquidity_pool_id_strkey`, but the Parquet file has no such column.

## Target Code

- `internal/transform/trade.go:80-98` — computes both raw and strkey pool identifiers for LP claims
- `internal/transform/trade.go:131-157` — stores `SellingLiquidityPoolIDStrkey` on `TradeOutput`
- `internal/transform/schema.go:285-312` — JSON trade schema declares `selling_liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:218-244` — Parquet trade schema omits the field
- `internal/transform/parquet_converter.go:248-275` — `ToParquet()` omits the field
- `cmd/export_trades.go:55-70` — trade Parquet export path writes `TradeOutputParquet`

## Evidence

For LP claim atoms, `TransformTrade()` encodes the pool ID to strkey and stores it in `SellingLiquidityPoolIDStrkey` (`internal/transform/trade.go:85-92`, `:150-156`). Trade tests explicitly expect populated strkey values on LP trades (`internal/transform/trade_test.go:789`, `internal/transform/trade_test.go:815`). The Parquet schema and converter never mention the field (`internal/transform/schema_parquet.go:218-244`, `internal/transform/parquet_converter.go:248-275`).

## Anti-Evidence

Parquet still preserves `selling_liquidity_pool_id` in its raw string form, so the row retains one pool identifier. But the transform already treats the strkey as a separate exported column, and Parquet provides no equivalent field for consumers that rely on that canonical representation.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-16
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of ai-summary/success/data-transform/005-trade-parquet-drops-liquidity-pool-strkey.md.gh-published
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same finding already confirmed and published in `ai-summary/success/data-transform/005-trade-parquet-drops-liquidity-pool-strkey.md.gh-published`. Both identify the missing `SellingLiquidityPoolIDStrkey` field in `TradeOutputParquet`, the same root cause in the Parquet schema and converter, and the same affected code paths.

### Code Paths Examined

- `ai-summary/success/data-transform/005-trade-parquet-drops-liquidity-pool-strkey.md.gh-published` — previously confirmed finding with identical mechanism, root cause, and affected code references

### Why It Failed

This is a re-discovery of an already-confirmed and published finding. The success file documents the same bug: `TradeOutputParquet` lacks `SellingLiquidityPoolIDStrkey`, causing LP trade Parquet exports to silently drop the canonical pool strkey identifier. The target code references, mechanism, and expected behavior are substantively identical.

### Lesson Learned

Check the success directory for matching findings before investing in full code tracing — especially when hypothesis numbering matches an existing success entry.
