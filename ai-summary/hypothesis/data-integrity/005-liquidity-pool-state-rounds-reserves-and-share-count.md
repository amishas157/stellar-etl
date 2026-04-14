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
