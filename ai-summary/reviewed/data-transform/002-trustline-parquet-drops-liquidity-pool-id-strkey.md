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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformTrustline()` (trustline.go:43-50) calls `strkey.Encode(strkey.VersionByteLiquidityPool, xdrPoolID[:])` for pool-share trustlines and assigns the result to `TrustlineOutput.LiquidityPoolIDStrkey` at line 87. The `TrustlineOutput` struct (schema.go:257) declares this field with JSON tag `liquidity_pool_id_strkey`. However, `TrustlineOutputParquet` (schema_parquet.go:172-191) has exactly 18 fields — every field from `TrustlineOutput` except `LiquidityPoolIDStrkey`. The `ToParquet()` converter (parquet_converter.go:199-219) correspondingly maps all 18 fields but never references `LiquidityPoolIDStrkey`. The field is silently dropped in every Parquet export of pool-share trustlines.

### Code Paths Examined

- `internal/transform/trustline.go:43-50` — `strkey.Encode()` call produces a valid `L...` strkey for pool-share trustlines; assigned to output at line 87
- `internal/transform/schema.go:237-258` — `TrustlineOutput` has 19 fields including `LiquidityPoolIDStrkey string` at line 257
- `internal/transform/schema_parquet.go:172-191` — `TrustlineOutputParquet` has 18 fields; `LiquidityPoolIDStrkey` is absent
- `internal/transform/parquet_converter.go:199-219` — `ToParquet()` maps 18 fields; no `LiquidityPoolIDStrkey` assignment exists
- `internal/transform/trustline_test.go:165` — Test fixture expects `LiquidityPoolIDStrkey: "LAAQGBAFA4EQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA2VM"`, confirming the JSON value is populated

### Findings

This is an exact instance of Investigation Pattern 2 (Parquet field mapping). The `TrustlineOutputParquet` struct was defined with 18 of 19 fields from `TrustlineOutput`, omitting only `LiquidityPoolIDStrkey`. The `ToParquet()` converter correspondingly has no line for the missing field. This is the same bug pattern as the already-confirmed `success/data-transform/002-contract-event-parquet-operation-id-dropped.md` and the sibling finding in `reviewed/data-transform/001-liquidity-pool-parquet-drops-pool-id-strkey.md` (which affects `PoolOutput`, a different entity). This finding affects `TrustlineOutput` — a distinct code path and output type.

The impact is that any downstream consumer reading Parquet exports (e.g., BigQuery analytics) that joins or filters on the canonical `L...` strkey identifier for pool-share trustlines will find the column entirely absent, while the same data exported as JSON contains the value. This silently breaks schema parity between the two export formats.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go` (append to existing if present, otherwise create)
- **Setup**: Construct a `TrustlineOutput` with a non-empty `LiquidityPoolIDStrkey` value (e.g., `"LAAQGBAFA4EQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA2VM"` from the existing test fixture at `trustline_test.go:165`)
- **Steps**: Call `trustlineOutput.ToParquet()` and type-assert the result to `TrustlineOutputParquet`
- **Assertion**: Verify that `TrustlineOutputParquet` has no `LiquidityPoolIDStrkey` field (the struct itself lacks it), confirming the data is silently dropped. Use reflection to show the JSON struct has the field but the Parquet struct does not.
