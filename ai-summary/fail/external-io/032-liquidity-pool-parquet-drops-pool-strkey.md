# H001: Liquidity-pool Parquet drops `liquidity_pool_id_strkey`

**Date**: 2026-04-15
**Subsystem**: external-io
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-pools --write-parquet` encounters a liquidity-pool ledger change, the Parquet row should preserve the same canonical `L...` StrKey identifier that `TransformPool()` already computes for the JSON row. Downstream Parquet consumers should be able to join on either `liquidity_pool_id` or `liquidity_pool_id_strkey` exactly as they can with the JSON export.

## Mechanism

`TransformPool()` populates `PoolOutput.PoolIDStrkey`, but the parallel Parquet schema never added a `PoolIDStrkey` column and `PoolOutput.ToParquet()` never copies the value. The export therefore writes a plausible Parquet row that keeps only the hex pool ID while silently discarding the canonical StrKey representation from every affected row.

## Trigger

Run `export_ledger_entry_changes --export-pools --write-parquet` over any ledger range containing at least one `LedgerEntryTypeLiquidityPool` change. Compare a JSON row and the corresponding Parquet schema/value set: JSON includes a non-empty `liquidity_pool_id_strkey`, while the Parquet row has no column that can carry it.

## Target Code

- `internal/transform/liquidity_pool.go:TransformPool:60-88` — computes `poolIDStrkey` and stores it on `PoolOutput`
- `internal/transform/schema.go:PoolOutput:203-225` — JSON/output schema exposes `liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:PoolOutputParquet:137-159` — Parquet schema omits the StrKey field entirely
- `internal/transform/parquet_converter.go:PoolOutput.ToParquet:163-185` — converter copies the hex pool ID but has no StrKey mapping
- `cmd/export_ledger_entry_changes.go:exportTransformedData:348-351,370-372` — production path selects `PoolOutputParquet` and writes only `ToParquet()` output

## Evidence

The transform path already treats the StrKey as part of the exported pool record: `TransformPool()` derives it with `strkey.Encode(...)` and stores it on `PoolOutput.PoolIDStrkey`. The omission happens only at the JSON-to-Parquet boundary, where both the row type and the converter lack any destination field.

## Anti-Evidence

The remaining `liquidity_pool_id` hex string still identifies the pool, so some consumers may not notice the loss immediately. But the repository already exports the StrKey as a first-class field in JSON, which means the Parquet drop is a real format divergence rather than an intentional schema simplification.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-transform/008-liquidity-pool-parquet-drops-pool-id-strkey.md.gh-published`
**Failed At**: reviewer

### Trace Summary

The hypothesis correctly identifies that `PoolOutputParquet` (schema_parquet.go:137-159) omits the `PoolIDStrkey` field present in `PoolOutput` (schema.go:224), and that `ToParquet()` (parquet_converter.go:163-185) therefore cannot carry the value. The bug is real and the code path is confirmed. However, this exact finding has already been confirmed and published under the `data-transform` subsystem.

### Code Paths Examined

- `internal/transform/liquidity_pool.go:TransformPool:60-88` — confirmed `poolIDStrkey` computed via `strkey.Encode` and stored on `PoolOutput.PoolIDStrkey`
- `internal/transform/schema.go:PoolOutput:203-225` — confirmed `PoolIDStrkey string` field at line 224
- `internal/transform/schema_parquet.go:PoolOutputParquet:137-159` — confirmed no `PoolIDStrkey` field
- `internal/transform/parquet_converter.go:PoolOutput.ToParquet:163-185` — confirmed no `PoolIDStrkey` assignment

### Why It Failed

This is an exact duplicate of `ai-summary/success/data-transform/008-liquidity-pool-parquet-drops-pool-id-strkey.md.gh-published`, which identifies the same root cause (missing `PoolIDStrkey` in `PoolOutputParquet` and `ToParquet()`), same affected code paths, and same impact. The only difference is the subsystem label (`external-io` vs `data-transform`).

### Lesson Learned

Cross-subsystem novelty checks are essential. The "Parquet drops strkey" pattern for liquidity pools was already confirmed via the `data-transform` subsystem. Always search success directories across ALL subsystems (not just the target) before filing.
