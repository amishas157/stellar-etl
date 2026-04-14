# H002: `liquidity_pool_withdraw` rounds share burn and realized reserve amounts

**Date**: 2026-04-14
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A successful `liquidity_pool_withdraw` row should preserve the exact burned share amount and exact withdrawn reserve amounts derived from the ledger delta. If two valid withdrawals differ by 1 stroop in `shares` or in either realized reserve amount, the exported `details.shares`, `details.reserve_a_withdraw_amount`, and `details.reserve_b_withdraw_amount` should remain different.

## Mechanism

The live withdraw branch converts `op.Amount`, `receivedA`, and `receivedB` through `utils.ConvertStroopValueToReal()`, so large successful withdrawals are rounded to the nearest `float64`. The duplicate formatter in the same file keeps these same values exact with `amount.String(...)`, which means the live exporter is discarding precision that is still available after `getLiquidityPoolAndProductDelta()` computes the exact `Int64` reserve deltas.

## Trigger

Export two otherwise identical successful `liquidity_pool_withdraw` operations whose burned shares or realized reserve deltas differ by 1 stroop at magnitudes above float64's exact decimal precision, e.g. a `shares` value of `90071992547409930` versus `90071992547409931`. Compare `details["shares"]` or either `reserve_*_withdraw_amount`: the current exporter should collapse the rows to the same numeric output.

## Target Code

- `internal/transform/operation.go:1047-1061` — live withdraw branch writes `reserve_*_withdraw_amount` and `shares` via `ConvertStroopValueToReal`
- `internal/transform/operation.go:1676-1682` — sibling formatter preserves `shares` and realized reserves with `amount.String(...)`
- `internal/utils/main.go:84-87` — stroop conversion ends in `float64`

## Evidence

The live path computes exact reserve deltas from the liquidity-pool ledger change and then immediately rounds them when populating operation details. The duplicate formatter proves the same data can be carried forward losslessly as decimal strings, so this is not an unavoidable consequence of the upstream XDR.

## Anti-Evidence

The already-published LP bound findings cover `reserve_*_min_amount`, not the realized withdraw amounts or burned shares targeted here. Small withdrawals still serialize correctly; the bug requires large values whose 7-decimal representation exceeds float64 precision.
