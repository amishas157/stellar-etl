# H005: liquidity-pool state rounds exact reserves and pool-share supply

**Date**: 2026-04-14
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Liquidity-pool state export should preserve the exact reserve balances and total pool-share supply stored on-chain. Two valid pool ledger entries that differ by 1 stroop in `ReserveA`, `ReserveB`, or `TotalPoolShares` should produce different exported `asset_a_amount`, `asset_b_amount`, or `pool_share_count` values.

## Mechanism

`TransformPool()` converts `cp.ReserveA`, `cp.ReserveB`, and `cp.TotalPoolShares` from exact XDR `Int64` values into `float64` via `utils.ConvertStroopValueToReal()`. Once those quantities are large enough, distinct pool states collapse to the same exported numbers, which silently corrupts reserve accounting even though the source ledger entry still contains the exact stroop totals.

## Trigger

Run `TransformPool()` on two otherwise identical liquidity-pool ledger entries whose `ReserveA` or `TotalPoolShares` differ by 1 stroop above float64's exact decimal precision threshold, such as `90071992547409930` versus `90071992547409931`. Compare `asset_a_amount` or `pool_share_count`: the current JSON rows should serialize the same rounded value for both states.

## Target Code

- `internal/transform/liquidity_pool.go:66-81` — writes `PoolShareCount`, `AssetAReserve`, and `AssetBReserve` via `ConvertStroopValueToReal`
- `internal/transform/schema.go:202-216` — `PoolOutput` exposes the rounded reserve/share columns
- `internal/utils/main.go:84-87` — shared stroop helper returns `float64`
- `.../xdr/xdr_generated.go:7239-7244` — liquidity-pool reserves and total shares are exact XDR `Int64`

## Evidence

The pool transform takes exact ledger-entry quantities and loses precision only at the final ETL mapping step. These columns are authoritative state snapshots used for reserve analytics and invariant checks, so rounding away a stroop changes the exported financial state rather than merely formatting it.

## Anti-Evidence

The issue is magnitude-dependent and will not show up for typical small pools. The existing liquidity-pool findings in `ai-summary` cover missing identifiers and operation-detail rounding, not the persistent reserve/share columns of the pool-state table.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete code path from `TransformPool()` (liquidity_pool.go:66-88) through `utils.ConvertStroopValueToReal()` (utils/main.go:85-87). Confirmed that `ConvertStroopValueToReal` converts exact `xdr.Int64` values to `float64` via `big.NewRat(int64(input), 10000000).Float64()`, which is single-rounding but still lossy for stroop values exceeding 2^53 (~9×10^15). The three affected fields (`PoolShareCount`, `AssetAReserve`, `AssetBReserve`) are stored as `float64` in both JSON (`PoolOutput`) and Parquet (`PoolOutputParquet`) schemas with no companion raw/exact field, making the precision loss permanent and unrecoverable.

### Code Paths Examined

- `internal/transform/liquidity_pool.go:66-88` — `TransformPool()` constructs `PoolOutput` with `PoolShareCount: utils.ConvertStroopValueToReal(cp.TotalPoolShares)`, `AssetAReserve: utils.ConvertStroopValueToReal(cp.ReserveA)`, `AssetBReserve: utils.ConvertStroopValueToReal(cp.ReserveB)`. All three fields convert exact `xdr.Int64` to lossy `float64`.
- `internal/utils/main.go:85-87` — `ConvertStroopValueToReal(input xdr.Int64) float64` uses `big.NewRat(int64(input), int64(10000000)).Float64()`. This is single-rounding (best possible), but still loses precision for inputs > 2^53.
- `internal/transform/schema.go:202-225` — `PoolOutput` defines `PoolShareCount float64`, `AssetAReserve float64`, `AssetBReserve float64` with no companion integer/string exact fields.
- `internal/transform/schema_parquet.go:137-159` — `PoolOutputParquet` mirrors the same `float64` types (`DOUBLE` parquet type) for all three fields.
- `internal/transform/parquet_converter.go:163-186` — `PoolOutput.ToParquet()` copies `float64` values directly to Parquet struct with no additional precision.
- XDR source (`xdr_generated.go:7241-7243`) — `ReserveA Int64`, `ReserveB Int64`, `TotalPoolShares Int64` are all exact 64-bit integers.

### Findings

**Confirmed**: The mechanism is identical to the already-published success/data-integrity/024-change-trust-limit-float64-rounding finding, applied to a different export surface (pool state table vs. operation details). The `ConvertStroopValueToReal` function converts exact `xdr.Int64` reserves and shares to `float64`, losing single-stroop precision for values > 2^53 stroops (~900 million XLM equivalent).

**Key distinctions from existing findings**:
1. **Different export table**: This affects the `liquidity_pools` state table, not `history_operations.details` (024) or `history_trades` (003-reviewed) or `offers` (004-reviewed).
2. **Three affected fields**: `pool_share_count`, `asset_a_amount`, and `asset_b_amount` are all lossy.
3. **No companion exact field**: Unlike token transfers (which have `amount_raw`) or prices (which have `*_price_r`), pool state has NO fallback exact representation. The `float64` is the only exported value, making the data loss permanent.
4. **State snapshots vs. events**: Pool reserves are cumulative state snapshots used for reserve accounting and invariant checking. Corrupted reserves can cascade into derived analytics (TVL calculations, impermanent loss estimates, pool share valuations).

**Severity rationale**: Critical. Pool reserves and share counts are primary financial data. The absence of any companion exact field means downstream consumers cannot recover the true value. The threshold (~900M XLM or equivalent asset units) is within the feasible range for large issued-asset pools, though unlikely for XLM-only pools.

### PoC Guidance

- **Test file**: `internal/transform/liquidity_pool_test.go`
- **Setup**: Create two `ingest.Change` objects containing liquidity pool entries with `ReserveA` values of `90071992547409930` and `90071992547409931` (adjacent stroops above 2^53). Use identical pool parameters for everything else.
- **Steps**: Call `TransformPool()` on both changes. Compare the resulting `PoolOutput.AssetAReserve` values. Also serialize both outputs with `json.Marshal` and compare the `asset_a_amount` JSON fields.
- **Assertion**: Assert that both `AssetAReserve` float64 values are equal (demonstrating the collision). Assert that both serialized JSON `asset_a_amount` values are identical strings despite different source stroops. Repeat for `PoolShareCount` with `TotalPoolShares` values of `90071992547409930` vs `90071992547409931`.
