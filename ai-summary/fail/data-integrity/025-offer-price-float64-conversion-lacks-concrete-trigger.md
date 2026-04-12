# H025: `offers.price` loses distinct on-chain prices through float64 conversion

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the `offers.price` column cannot preserve the exact `price_n / price_d`
ratio, there should be a concrete pair of distinct valid on-chain prices whose
exported `price` values collide or materially diverge in live output.

## Mechanism

`TransformOffer()` computes `OfferOutput.Price` as
`float64(outputPriceN) / float64(outputPriceD)`. That looked suspicious because
the exact rational inputs are available at the same time, and downstream users
may rely on the scalar `price` column instead of reconstructing from
`price_n` / `price_d`.

## Trigger

1. Construct two valid offer entries with distinct `xdr.Price{N,D}` pairs.
2. Export them through `TransformOffer()`.
3. Compare the resulting scalar `price` values for any float64 collision or
   materially wrong approximation.

## Target Code

- `internal/transform/offer.go:49-66` — computes `PriceN`, `PriceD`, and the
  scalar `Price` float64 from the same exact rational source
- `internal/transform/schema.go:269-281` — exports all three fields on the row

## Evidence

The code does narrow an exact rational into a float64 column, so the suspicion
was reasonable on first read. The exact numerator and denominator are also
exported, which means the table mixes an approximate scalar with exact sibling
fields.

## Anti-Evidence

I did not establish a concrete live trigger in the current int32 price domain.
Exploratory search over a large sample of valid `N/D` pairs did not produce a
distinct pair that collapsed to the same exported float64, and unlike prior
viable findings this path is already using the best available float64
representation rather than an unnecessarily lossy string round-trip.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

No concrete trigger was established showing two distinct valid `xdr.Price`
values producing the same or materially wrong `offers.price` output. Without a
reproducible collision or divergence, this remains a generic float-schema
concern rather than a demonstrated ETL bug.

### Lesson Learned

For float64 hypotheses on rational price fields, do not stop at "exact inputs
were narrowed to float." A viable finding needs an explicit pair of reachable
on-chain values that actually collide or export the wrong scalar in the current
domain, especially when the exact numerator/denominator fields are already
preserved alongside the float.
