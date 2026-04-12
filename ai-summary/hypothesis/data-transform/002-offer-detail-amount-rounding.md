# H002: Sell-offer family detail amounts round large order sizes

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For successful `manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` operations, `history_operations.details.amount` should preserve the exact stroop-denominated order quantity. Two offers whose `BuyAmount`/`Amount` differ by 1 stroop should export different detail values rather than the same rounded `float64`.

## Mechanism

All three offer-family branches write `details["amount"]` via `utils.ConvertStroopValueToReal()`, which rounds large 7-decimal amounts to the nearest binary float. In the same file, the sibling formatter already uses `amount.String(...)` for these exact operation fields, and the live output map already mixes strings and numbers, so the precision loss is caused by this branch-local conversion choice rather than by a schema constraint.

## Trigger

1. Build successful `manage_buy_offer`, `manage_sell_offer`, or `create_passive_sell_offer` operations with `BuyAmount`/`Amount` set to `90071992547409930` stroops and `90071992547409931` stroops.
2. Run each through `TransformOperation()`.
3. Compare `OperationDetails["amount"]` across the two rows; the exact decimal strings differ, but the exported JSON number can collapse.

## Target Code

- `internal/transform/operation.go:701-755` — live offer-family detail export converts `amount` to `float64`
- `internal/transform/operation.go:1424-1455` — sibling formatter keeps exact `amount.String(...)` values for the same offer branches
- `internal/transform/schema.go:136-150` — operation details are emitted through `map[string]interface{}`

## Evidence

`manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` all populate `details["amount"]` from `utils.ConvertStroopValueToReal(...)`. The alternate formatter in the same file uses exact decimal strings for those same fields, which shows an exact representation is already available and consistent with this output surface.

## Anti-Evidence

The offer-family rows also emit exact `price_r` numerators and denominators, so price reconstruction still works. But there is no exact sibling for `amount`, so the requested order size itself is silently corrupted on the decoded operation-details surface.
