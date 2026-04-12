# H022: Trustline Parquet drops `liquidity_pool_id_strkey`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Pool-share trustline rows should preserve both the raw liquidity-pool ID and the canonical `L...` StrKey form in Parquet, just as they do in JSON. The two fields are not interchangeable unless downstream systems re-implement the same encoding logic.

## Mechanism

`TransformTrustline()` detects pool-share trustlines, derives `LiquidityPoolIDStrkey`, and stores it on `TrustlineOutput`. The ledger-entry-change Parquet path then serializes `TrustlineOutput` through a Parquet schema that has no `liquidity_pool_id_strkey` column and a converter that never copies it, so every pool-share trustline loses that identifier in Parquet only.

## Trigger

Run `export_ledger_entry_changes --export-trustlines --write-parquet` on any ledger range containing a pool-share trustline. The JSON row will contain `liquidity_pool_id_strkey`, but the corresponding Parquet row will not.

## Target Code

- `internal/transform/trustline.go:TransformTrustline:43-50` — encodes the liquidity-pool ID into StrKey form for pool-share trustlines
- `internal/transform/trustline.go:TransformTrustline:68-88` — assigns `LiquidityPoolIDStrkey` onto the exported trustline row
- `internal/transform/schema.go:TrustlineOutput:237-258` — exposes `liquidity_pool_id_strkey` in the schema
- `internal/transform/schema_parquet.go:TrustlineOutputParquet:171-190` — defines no Parquet column for that field
- `internal/transform/parquet_converter.go:TrustlineOutput.ToParquet:199-219` — omits the StrKey field entirely
- `cmd/export_ledger_entry_changes.go:exportTransformedData:356-372` — writes the lossy Parquet rows for trustline exports

## Evidence

The trustline transform only computes `LiquidityPoolIDStrkey` on the pool-share branch, which shows it is intended to carry extra semantics beyond the raw hex pool ID. `internal/transform/trustline_test.go:150-166` expects a populated `LiquidityPoolIDStrkey` for a pool-share fixture. That field disappears only on the Parquet path.

## Anti-Evidence

As with trades, the raw `liquidity_pool_id` survives in Parquet. But that does not preserve JSON/Parquet parity, and hashes/hex IDs are not a drop-in substitute for the canonical StrKey form that the transform already emits.
