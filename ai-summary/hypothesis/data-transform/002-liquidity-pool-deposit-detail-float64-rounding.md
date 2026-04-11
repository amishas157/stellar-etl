# H002: Liquidity-pool deposit details round large reserve deltas and share amounts

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a successful `liquidity_pool_deposit` operation, the exported detail fields `reserve_a_deposit_amount`, `reserve_b_deposit_amount`, and `shares_received` should preserve the exact decimal values implied by the underlying `xdr.Int64` reserve/share deltas. A one-stroop difference in the pool deltas should remain visible in the operation JSON.

## Mechanism

The deposit detail builder first formats the exact `xdr.Int64` amounts as decimal strings with `amount.String(...)`, then immediately parses those strings back into `float64` with `strconv.ParseFloat(..., 64)`. For large deltas, the float64 parse rounds away low-order digits, so operation details publish plausible-but-wrong deposit and share values even though the preceding string conversion had the exact decimal representation available.

## Trigger

Export a successful `liquidity_pool_deposit` whose reserve delta or `TotalPoolShares` delta exceeds float64's exact-integer range after 7-decimal scaling, such as a deposit delta of `9007199254740993` stroops. The detail map will report `reserve_*_deposit_amount` or `shares_received` as the rounded neighbor value instead of the exact amount.

## Target Code

- `internal/transform/operation.go:974-1019` — successful deposit path computes deltas, then parses `amount.String(...)` back into `float64`
- `internal/transform/operation.go:238-285` — `getLiquidityPoolAndProductDelta()` returns exact `xdr.Int64` reserve/share deltas before the rounding step
- `internal/transform/operation_test.go:1751-1804` — tests cover only tiny deposit/share values

## Evidence

The code already has the exact decimal strings in hand; the only reason precision is lost is the final `ParseFloat` step. This is a live transform-layer bug in the JSON operation export path, not a Parquet-only issue, and it affects three separate monetary fields on the same operation row.

## Anti-Evidence

Typical test fixtures use tiny reserve deltas, so the exported values look correct in current coverage. If downstream consumers ignore these float fields and rebuild amounts from raw XDR elsewhere, they can recover, but the exported operation details themselves are still corrupted.
