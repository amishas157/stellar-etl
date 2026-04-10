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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformPool()` (liquidity_pool.go:60-64) calls `strkey.Encode(strkey.VersionByteLiquidityPool, lp.LiquidityPoolId[:])` and assigns the result to `PoolOutput.PoolIDStrkey` at line 87. The `PoolOutput` struct (schema.go:224) declares this field with JSON tag `liquidity_pool_id_strkey`. However, `PoolOutputParquet` (schema_parquet.go:138-159) has exactly 19 fields — every field from `PoolOutput` except `PoolIDStrkey`. The `ToParquet()` converter (parquet_converter.go:163-185) correspondingly maps all 19 fields but never references `PoolIDStrkey`. The field is silently dropped in every Parquet export.

### Code Paths Examined

- `internal/transform/liquidity_pool.go:60-64` — `strkey.Encode()` call produces a valid `L...` strkey; assigned to output at line 87
- `internal/transform/schema.go:202-225` — `PoolOutput` has 20 fields including `PoolIDStrkey string` at line 224
- `internal/transform/schema_parquet.go:138-159` — `PoolOutputParquet` has 19 fields; `PoolIDStrkey` is absent
- `internal/transform/parquet_converter.go:163-185` — `ToParquet()` maps 19 fields; no `PoolIDStrkey` assignment exists
- `internal/transform/liquidity_pool_test.go:119` — Test fixture expects `PoolIDStrkey: "LALS2QYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAC2X"`, confirming the JSON value is populated

### Findings

This is an exact instance of Investigation Pattern 2 (Parquet field mapping). The `PoolOutputParquet` struct was defined with 19 of 20 fields from `PoolOutput`, omitting only `PoolIDStrkey`. The `ToParquet()` converter correspondingly has no line for the missing field. This is the same bug pattern as the already-confirmed finding `success/data-transform/002-contract-event-parquet-operation-id-dropped.md`, where `ContractEventOutputParquet` omitted `OperationID`.

The impact is that any downstream consumer reading Parquet exports (e.g., BigQuery analytics) that joins or filters on the canonical `L...` strkey identifier will find the column entirely absent, while the same data exported as JSON contains the value. This silently breaks schema parity between the two export formats.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go` (append to existing if present, otherwise create)
- **Setup**: Construct a `PoolOutput` with a non-empty `PoolIDStrkey` value (e.g., `"LALS2QYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAC2X"` from the existing test fixture)
- **Steps**: Call `poolOutput.ToParquet()` and type-assert the result to `PoolOutputParquet`
- **Assertion**: Verify that `PoolOutputParquet` has no `PoolIDStrkey` field (the struct itself lacks it), confirming the data is silently dropped. Alternatively, compare the field count or use reflection to show the JSON struct has the field but the Parquet struct does not.
