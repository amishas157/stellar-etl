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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not a direct duplicate, but substantially equivalent to 034-account-export-float64-collides-adjacent-stroop-balances
**Failed At**: reviewer

### Trace Summary

Traced `TransformTrade()` in `internal/transform/trade.go:139,145` which converts `SellingAmount` and `BuyingAmount` through `utils.ConvertStroopValueToReal()`. This helper at `internal/utils/main.go:85-87` uses `big.NewRat(int64(input), int64(10000000)).Float64()`, which produces the nearest representable float64 — the best possible conversion within the float64 type. The `TradeOutput` schema at `internal/transform/schema.go:294,300` defines both amount fields as `float64` with no companion raw stroop field. No alternative conversion path exists in the trade transform.

### Code Paths Examined

- `internal/transform/trade.go:TransformTrade:139` — `SellingAmount: utils.ConvertStroopValueToReal(outputSellingAmount)` uses the shared helper
- `internal/transform/trade.go:TransformTrade:145` — `BuyingAmount: utils.ConvertStroopValueToReal(xdr.Int64(outputBuyingAmount))` uses the shared helper
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — confirmed `big.NewRat(...).Float64()` produces the nearest float64 (best possible conversion)
- `internal/transform/schema.go:TradeOutput:286-312` — confirmed `SellingAmount float64` and `BuyingAmount float64` with no raw stroop companion field

### Why It Failed

The hypothesis describes an inherent precision limitation of the `float64` schema type, not a coding bug. The trade transform correctly uses `ConvertStroopValueToReal()`, which uses `big.NewRat().Float64()` to produce the nearest representable float64 — the best possible conversion within the float64 type. Unlike the confirmed findings where a better path existed but wasn't used (016-claimable-balance used an inferior inline `float64(amount)/1.0e7` instead of the shared helper; 017-LP-deposit discarded an exact `amount.String()` by parsing it back through `strconv.ParseFloat`), the trade transform has no "better path available but not used." The conversion is already optimal within the schema's type constraint. This is the same analysis that led to the rejection of 034-account-export-float64-collides-adjacent-stroop-balances, applied to a different entity.

### Lesson Learned

Float64 precision-loss hypotheses for entities using `ConvertStroopValueToReal()` are inherent schema design limitations, not conversion bugs. Only entities that bypass the shared helper (016) or unnecessarily round-trip through string→float64 (017) qualify as bugs. The trade transform already uses the established correct path.
