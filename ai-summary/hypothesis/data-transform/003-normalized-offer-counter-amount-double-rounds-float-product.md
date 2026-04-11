# H003: Normalized-offer `counter_amount` is derived from a double-rounded float product

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`dim_offers.counter_amount` should be derived from the exact on-chain offer amount and exact rational price, then rounded once at the final `float64` boundary. A large offer with `Amount = 9007199254740993` stroops and `Price = 123456789/10000000` should export `counter_amount = 11119998978.73516`, which is the `float64` representation of the exact rational product.

## Mechanism

`TransformOffer()` first rounds the raw stroop amount into `offer.Amount float64` and the rational price into `offer.Price float64`. `extractDimOffer()` then multiplies those two rounded floats (`float64(offer.Amount) * offer.Price`) instead of multiplying the original stroop integer by `PriceN/PriceD`. That double-rounding compounds error: for the trigger above, the current code exports `11119998978.735159` while the exact-rational-then-float path exports `11119998978.73516`, a drift of about 19 stroops.

## Trigger

Process any normalized offer with a large amount plus a non-trivial rational price, for example:

1. `OfferEntry.Amount = 9007199254740993`
2. `Price.N = 123456789`
3. `Price.D = 10000000`

Under the current code:

- `offer.Amount` becomes `900719925.4740993`
- `offer.Price` becomes `12.3456789`
- `counter_amount` exports as `11119998978.735159`

But the exact rational calculation `float((Amount / 1e7) * (N / D))` yields `11119998978.73516`.

## Target Code

- `internal/transform/offer.go:44-66,79-101` — converts raw offer amount and price into float64 fields before normalization
- `internal/transform/offer_normalized.go:139-167` — multiplies `offer.Amount` and `offer.Price` floats to derive `CounterAmount`
- `internal/transform/schema.go:321-329` — `DimOffer.CounterAmount` has no exact companion field, so the drift is unrecoverable from the normalized row

## Evidence

A brute-force search over valid `int64` amounts and `int32/int32` price fractions produces concrete divergences between the current float-product path and an exact rational product converted once at the end. The trigger above is one such case, and the discrepancy is larger than a single stroop, so downstream orderbook analytics can misstate quote-side size even when no hash collision occurs.

## Anti-Evidence

The output schema is still `float64`, so extremely large products will always have some precision limits. This hypothesis is narrower and testable: it targets an avoidable extra error from multiplying two already-rounded floats even though the exact rational inputs (`Amount`, `PriceN`, `PriceD`) are available earlier in the transform path.
