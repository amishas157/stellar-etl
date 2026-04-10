# H001: Liquidity-pool Parquet drops populated `liquidity_pool_id_strkey`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a liquidity-pool ledger entry is transformed, the Parquet row should preserve the same canonical `liquidity_pool_id_strkey` value that the JSON row exports. A row that already contains both the hex pool ID and the `L...` strkey in JSON should not lose the strkey solely because the export format changes to Parquet.

## Mechanism

`TransformPool()` always computes `PoolIDStrkey` with `strkey.Encode(...)` and stores it on `PoolOutput`, and the checked-in fixture already expects a non-empty `L...` value. But `PoolOutputParquet` has no `PoolIDStrkey` field and `PoolOutput.ToParquet()` stops at `LedgerSequence`, so the Parquet schema cannot represent the populated strkey at all. Every `export_ledger_entry_changes --write-parquet` run therefore silently drops a real identifier column from liquidity-pool rows.

## Trigger

Run `export_ledger_entry_changes --write-parquet` over any ledger range containing liquidity-pool ledger-entry changes. Compare the JSON and Parquet outputs for the same row: JSON includes `liquidity_pool_id_strkey`, while the Parquet file has no corresponding column.

## Target Code

- `internal/transform/liquidity_pool.go:TransformPool:60-88` — computes `PoolIDStrkey` and stores it on `PoolOutput`
- `internal/transform/schema.go:PoolOutput:203-225` — JSON schema includes `liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:PoolOutputParquet:137-159` — Parquet schema omits `PoolIDStrkey`
- `internal/transform/parquet_converter.go:PoolOutput.ToParquet:163-185` — converter never copies the strkey
- `cmd/export_ledger_entry_changes.go:321-351` — parquet export path chooses `PoolOutputParquet` for liquidity-pool rows

## Evidence

`TransformPool()` encodes `lp.LiquidityPoolId[:]` to a strkey at `liquidity_pool.go:60-64` and assigns it at `liquidity_pool.go:66-88`. The fixture in `internal/transform/liquidity_pool_test.go:97-120` already expects `PoolIDStrkey: "LALS2QY..."`, proving the JSON-layer value is populated in normal code. The Parquet struct at `schema_parquet.go:137-159` has no field for it, and `ToParquet()` correspondingly has no assignment.

## Anti-Evidence

The hex `liquidity_pool_id` still survives in Parquet, so pool identity is not completely lost. But downstream systems that use the canonical `L...` identifier the transform already computes cannot recover it from the Parquet export.
