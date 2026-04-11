# H004: Liquidity-pool Parquet export drops `liquidity_pool_id_strkey`

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When liquidity-pool ledger entries are exported with `--write-parquet`, the Parquet row should retain the same canonical `liquidity_pool_id_strkey` that appears in JSON. Downstream consumers should be able to join Parquet pool rows directly to other strkey-keyed datasets without reconstructing identifiers from the hex hash.

## Mechanism

`TransformPool()` computes `PoolIDStrkey` and the pool tests assert a populated value, but `PoolOutputParquet` and `PoolOutput.ToParquet()` omit that field entirely. `export_ledger_entry_changes` still routes `PoolOutput` through `WriteParquet`, so the Parquet export silently strips the strkey while JSON keeps it.

## Trigger

Run `export_ledger_entry_changes --export-liquidity-pools --write-parquet` over any ledger containing liquidity-pool entries. The JSON output will contain `liquidity_pool_id_strkey`; the Parquet schema/file will not.

## Target Code

- `internal/transform/liquidity_pool.go:60-88` — generates and stores `PoolIDStrkey`
- `internal/transform/schema.go:203-225` — declares `liquidity_pool_id_strkey` on `PoolOutput`
- `internal/transform/schema_parquet.go:137-159` — defines `PoolOutputParquet` without the strkey field
- `internal/transform/parquet_converter.go:163-185` — converts pool rows to Parquet without copying `PoolIDStrkey`
- `cmd/export_ledger_entry_changes.go:321-371` — sends `PoolOutput` rows to the Parquet writer

## Evidence

`liquidity_pool_test.go` expects a concrete exported strkey (`LALS2QY...`) on the JSON-side struct, so this identifier is intentionally part of the public output contract. The export wiring explicitly uses `PoolOutputParquet`, making the loss reproducible on ordinary ledger-entry parquet exports.

## Anti-Evidence

The hex `liquidity_pool_id` still exists in Parquet, so consumers that are willing to re-encode it can recover the missing identifier. JSON exports are unaffected.
