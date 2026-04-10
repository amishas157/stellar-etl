# H002: Trustline Parquet drops populated `liquidity_pool_id_strkey`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For pool-share trustlines, the Parquet row should preserve the same `liquidity_pool_id_strkey` value that the JSON row exports. A trustline row that identifies its backing pool with both hex and `L...` forms in JSON should keep both identifiers in Parquet as well.

## Mechanism

`TransformTrustline()` detects `AssetTypeAssetTypePoolShare`, computes both `LiquidityPoolID` and `LiquidityPoolIDStrkey`, and stores them on `TrustlineOutput`. But `TrustlineOutputParquet` omits `LiquidityPoolIDStrkey`, and `TrustlineOutput.ToParquet()` only copies the hex pool ID and other common fields. Pool-share trustlines therefore lose their canonical strkey pool identifier only in the Parquet path.

## Trigger

Run `export_ledger_entry_changes --write-parquet` on any ledger range containing a pool-share trustline. The JSON row will include a non-empty `liquidity_pool_id_strkey`, while the Parquet row has no such column.

## Target Code

- `internal/transform/trustline.go:TransformTrustline:43-50` — computes the liquidity-pool strkey for pool-share trustlines
- `internal/transform/trustline.go:TransformTrustline:68-88` — stores `LiquidityPoolIDStrkey` on `TrustlineOutput`
- `internal/transform/schema.go:TrustlineOutput:237-258` — JSON schema includes `liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:TrustlineOutputParquet:171-191` — Parquet schema omits the strkey field
- `internal/transform/parquet_converter.go:TrustlineOutput.ToParquet:199-219` — converter never copies the strkey
- `cmd/export_ledger_entry_changes.go:356-358` — parquet export path routes trustline rows through `TrustlineOutputParquet`

## Evidence

The pool-share branch in `trustline.go:43-50` calls `strkey.Encode(strkey.VersionByteLiquidityPool, ...)`, and the output struct assignment at `trustline.go:68-88` persists that value. The fixture in `internal/transform/trustline_test.go:150-166` expects `LiquidityPoolIDStrkey: "LAAQGBA..."`. But the Parquet schema at `schema_parquet.go:171-191` ends at `LedgerSequence`, and the converter at `parquet_converter.go:199-219` has no `LiquidityPoolIDStrkey` assignment.

## Anti-Evidence

The hex `liquidity_pool_id` is still exported, so a downstream user can reconstruct pool identity with extra format-specific logic. But the transform explicitly exposes the canonical `L...` identifier in JSON, so silently dropping it from Parquet is still structural data loss.
