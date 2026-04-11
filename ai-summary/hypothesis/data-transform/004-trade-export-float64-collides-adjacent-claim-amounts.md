# H004: Trade export rounds adjacent claimed amounts to the same float64

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Two trades whose sold or bought amount differs by one stroop should export distinct `selling_amount` and `buying_amount` values. For example, a claim atom selling `10735946655003267` stroops should not serialize identically to one selling `10735946655003268` stroops.

## Mechanism

`TransformTrade()` converts both claimed amounts to `float64` through `ConvertStroopValueToReal()` before storing them in `TradeOutput`. Once the claim amount crosses the float precision threshold, adjacent stroop values collapse to the same decimal representation, so executed trade volume in `history_trades` becomes off by one or more stroops without any error.

## Trigger

Process an order-book or liquidity-pool trade whose `AmountSold()` or `AmountBought()` is at or above `10735946655003267` stroops. Compare two trades that differ by one stroop in the claim atom: both exports emit the same `selling_amount` or `buying_amount`.

## Target Code

- `internal/transform/trade.go:TransformTrade:52-157` - trade amounts are converted with `utils.ConvertStroopValueToReal()`.
- `internal/transform/schema.go:TradeOutput:286-312` - `selling_amount` and `buying_amount` are stored as `float64`.
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` - final float conversion step.

## Evidence

The trade path has no exact raw-amount companion field; the only exported trade amounts are the float64 values in `selling_amount` and `buying_amount`. The same reproduced collision pair (`10735946655003267` vs `10735946655003268`) therefore turns into identical trade volume despite coming from different XDR claim amounts.

## Anti-Evidence

Many historical trades are much smaller than the collision threshold, so the corruption only appears on unusually large claims. If downstream consumers always join back to raw XDR for exact settlement amounts, reviewers may classify this as limited-impact despite the wrong exported numeric field.
