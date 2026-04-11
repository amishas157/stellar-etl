# H005: Liquidity-pool withdraw details round large reserve and share amounts

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A successful `liquidity_pool_withdraw` row should preserve the exact 7-decimal values for `reserve_a_min_amount`, `reserve_b_min_amount`, `reserve_a_withdraw_amount`, `reserve_b_withdraw_amount`, and `shares`. A withdrawal of `90071992547409931` stroops should export the exact decimal `9007199254.7409931`, and 1-stroop differences should remain visible.

## Mechanism

In `extractOperationDetails()`, all five of these withdraw fields are emitted through `utils.ConvertStroopValueToReal()`. The sibling `transactionOperationWrapper.Details()` path for the same operation already exports exact `amount.String(...)` values inside `reserves_min`, `reserves_received`, and `shares`, so the lossy float64 encoding is not required by the operation-details contract and can silently collapse distinct reserve/share deltas.

## Trigger

Process a successful `liquidity_pool_withdraw` whose min amounts, received reserves, or withdrawn share amount are large, for example:

1. `LiquidityPoolWithdrawOp.Amount = 90071992547409931`, or
2. a pool delta where `receivedA` / `receivedB = 90071992547409931`

Compare the exported detail fields against the exact `amount.String(...)` values or against a second withdrawal that differs by 1 stroop; the float fields will round together.

## Target Code

- `internal/transform/operation.go:1021-1061` — withdraw branch converts reserve mins, received amounts, and `shares` through `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1650-1684` — sibling wrapper emits exact strings for `reserves_min`, `reserves_received`, and `shares`
- `internal/transform/schema.go:136-150` — operation details are stored in an untyped map, so exact strings are representable

## Evidence

This is the withdraw-side analog of the already-confirmed LP-deposit precision bug, but with a different branch and a different exported key set. The same file already contains an exact-string representation for withdraw details, which makes the float64 conversion a branch-local data-loss decision rather than an unavoidable limitation.

## Anti-Evidence

There is already a rejected investigation about failed LP operations defaulting reserve assets to `"native"`; this hypothesis is different because it targets successful large-value withdrawals and exact-value loss, not failed-operation placeholder semantics. Current fixtures use tiny pool deltas, so they do not exercise the precision boundary.
