# H075: Offer-family `price` looked lossy, but the same row already preserves exact rational components

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `history_operations.details.price` is meant to be the authoritative offer price, it should preserve the exact rational value implied by the XDR `Price` instead of rounding through `float64`. Two distinct offer prices that differ only beyond binary-float precision should remain distinguishable in the decoded details payload.

## Mechanism

`addPriceDetails()` parses `price.String()` into a `float64`, which can round exact rational prices. At first glance this looks like the same avoidable precision-loss pattern as the amount fields in `extractOperationDetails()`.

## Trigger

1. Build successful `manage_buy_offer`, `manage_sell_offer`, or `create_passive_sell_offer` operations with large `xdr.Price` numerators and denominators whose exact decimal values are distinguishable but not exactly representable as `float64`.
2. Run them through `TransformOperation()` and inspect `details.price`.

## Target Code

- `internal/transform/operation.go:409-420` — `addPriceDetails()` parses exact price strings into `float64`
- `internal/transform/operation.go:701-755` — live offer-family branches call `addPriceDetails()`

## Evidence

The live details payload stores `price` as a binary float generated from `price.String()`. That conversion can round exact rationals just like any other decimal-to-float path.

## Anti-Evidence

The same live output row always includes `price_r` with exact numerator and denominator components. Unlike the amount fields, the exact price is already preserved on the same row without decoding any external XDR blob.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Even if `details.price` rounds, the row simultaneously exports `details.price_r`, which preserves the exact rational price and makes the value recoverable without leaving the row. That makes this a convenience-field precision issue rather than a meaningful silent data-corruption bug.

### Lesson Learned

Precision-loss hypotheses are strongest when the rounded field is the only decoded representation on the row. When the same row already carries an exact rational sibling such as `price_r`, the rounded convenience field is much less likely to qualify as a viable finding.
