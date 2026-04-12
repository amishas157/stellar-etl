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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformTrustline()` in trustline.go computes `LiquidityPoolIDStrkey` via `strkey.Encode(strkey.VersionByteLiquidityPool, xdrPoolID[:])` at line 45 for pool-share trustlines and assigns it to the `TrustlineOutput` struct at line 87. The JSON serialization path correctly emits this field. However, `TrustlineOutputParquet` (schema_parquet.go:171-191) has no corresponding column, and `TrustlineOutput.ToParquet()` (parquet_converter.go:199-219) never copies the value. The export path at export_ledger_entry_changes.go:356-358 routes trustline output through this lossy Parquet conversion, silently dropping the field for every pool-share trustline row.

### Code Paths Examined

- `internal/transform/trustline.go:TransformTrustline:43-50` — Confirmed: computes `poolIDStrkey` via `strkey.Encode` for pool-share trustlines
- `internal/transform/trustline.go:TransformTrustline:68-88` — Confirmed: assigns `LiquidityPoolIDStrkey: poolIDStrkey` in the output struct
- `internal/transform/schema.go:TrustlineOutput:257` — Confirmed: field `LiquidityPoolIDStrkey string` with JSON tag `liquidity_pool_id_strkey` present
- `internal/transform/schema_parquet.go:TrustlineOutputParquet:171-191` — Confirmed: 18 fields present, `LiquidityPoolIDStrkey` is NOT among them
- `internal/transform/parquet_converter.go:TrustlineOutput.ToParquet:199-219` — Confirmed: copies 18 fields from `TrustlineOutput` to `TrustlineOutputParquet`, `LiquidityPoolIDStrkey` is NOT copied
- `cmd/export_ledger_entry_changes.go:356-358` — Confirmed: `TrustlineOutput` case routes through `TrustlineOutputParquet` schema for Parquet writes

### Findings

The bug is a straightforward omission: when `TrustlineOutputParquet` was defined, the `LiquidityPoolIDStrkey` field was not included. The `ToParquet()` converter correspondingly never copies it. This creates a JSON/Parquet parity gap where every pool-share trustline loses the canonical `L...` StrKey encoding of its liquidity pool ID in Parquet output. The raw hex `liquidity_pool_id` survives, but downstream consumers expecting StrKey-encoded identifiers (consistent with how other Stellar identifiers are encoded) will not find the field in Parquet at all.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` or `internal/transform/trustline_test.go`
- **Setup**: Create a `TrustlineOutput` with a non-empty `LiquidityPoolIDStrkey` value (e.g., from the existing pool-share test fixture in `trustline_test.go:150-166`)
- **Steps**: Call `.ToParquet()` on the `TrustlineOutput` and inspect the returned `TrustlineOutputParquet` struct
- **Assertion**: Assert that the `TrustlineOutputParquet` struct has a `LiquidityPoolIDStrkey` field containing the same value — this will fail with the current code because the field doesn't exist on the Parquet struct. The fix requires adding `LiquidityPoolIDStrkey string` with appropriate parquet tags to `TrustlineOutputParquet` and copying it in `ToParquet()`.
