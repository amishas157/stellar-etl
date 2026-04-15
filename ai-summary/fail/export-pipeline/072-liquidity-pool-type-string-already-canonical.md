# H072: Liquidity-pool exports use raw XDR enum names for `type`

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: liquidity-pool type labeling
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformPool()` should export the canonical pool type string used by the table contract, e.g. `constant_product`, rather than a generated XDR enum label like `LiquidityPoolTypeLiquidityPoolConstantProduct`.

## Mechanism

The transform pulls `poolType` from `xdr.LiquidityPoolTypeToString[lp.Body.Type]`, which initially looks suspicious because many generated XDR `String()` helpers return verbose enum identifiers. If this map exposed raw enum names, every pool row would carry a plausible but non-canonical `type`.

## Trigger

Export any liquidity-pool ledger change and inspect the `type` column in the JSON row. If the map returned raw generated labels, the output would diverge from the expected `constant_product` value.

## Target Code

- `internal/transform/liquidity_pool.go:34-38,66-68` — transform sources `PoolType` from `xdr.LiquidityPoolTypeToString`
- `internal/transform/liquidity_pool_test.go:100-118` — test fixture expects `PoolType: "constant_product"`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:4169-4171` — generic `String()` helper for `LiquidityPoolType`

## Evidence

The transform uses an XDR string map rather than a local canonicalization helper, which is a common source of raw-enum leakage in other codebases. The generated `String()` helper nearby also has the usual enum-name shape, so the concern looked plausible on first read.

## Anti-Evidence

The checked-in transform test already asserts `PoolType: "constant_product"` for a current constant-product pool row, proving the map returns the canonical table string in this case. There is no observed divergence between the live transform and the expected output contract.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated in `export-pipeline`

### Why It Failed

`xdr.LiquidityPoolTypeToString` already produces the canonical `"constant_product"` string that the export expects, and the local test pins that exact value. The suspected raw-enum leak does not occur on the current code path.

### Lesson Learned

Generated XDR enums are only suspicious if the specific helper or map used by the transform actually emits raw names. When a local unit test already asserts the canonical output string, the enum-label concern should be retired quickly.
