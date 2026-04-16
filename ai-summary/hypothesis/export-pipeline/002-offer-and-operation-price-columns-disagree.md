# H002: Offer-state and offer-operation price columns diverge for the same `xdr.Price`

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a `ManageBuyOffer`, `ManageSellOffer`, or `CreatePassiveSellOffer` operation creates or updates an offer, the scalar price exported in operation details should agree with the scalar price exported in the resulting `offers` state row for the same on-chain `xdr.Price`.

## Mechanism

The two live exports use different lossy conversions for the same source rational. `TransformOffer()` computes `offers.price` with raw `float64(N) / float64(D)`, while `addPriceDetails()` in the operation path calls `price.String()` and then `strconv.ParseFloat(...)`, which rounds the rational to 7 decimal places before parsing. That means the same on-chain price can export as two different numeric values across datasets: for `2147478646/2147478647`, operation details export `price = 1.0` while the offer-state row exports `price = 0.9999999995343376`; for `1/20000001`, operation details export `0` while the offer-state row stays non-zero.

## Trigger

Export a ledger containing an offer operation whose `Price` is either `2147478646/2147478647` or `1/20000001`, and also export the resulting offer state from `export_ledger_entry_changes`. Compare the scalar operation `details.price` with the scalar `offers.price` for the same rational price.

## Target Code

- `internal/transform/operation.go:409-420` — `addPriceDetails()` converts `xdr.Price` through `price.String()` and `strconv.ParseFloat`
- `internal/transform/operation.go:701-748` — live offer-operation branches that populate `details.price`
- `internal/transform/offer.go:63-66` — converts offer-state `Price` with direct `float64` division
- `internal/transform/offer.go:79-101` — writes the divergent scalar into `OfferOutput`

## Evidence

The current code has two separate scalar-price conversion pipelines with materially different rounding behavior. A direct reproduction of those two code paths on concrete rationals showed `2147478646/2147478647 -> 1.0` in operation details but `0.9999999995343376` in offer state, and `1/20000001 -> 0.0` in operation details but `4.999999750000013e-08` in offer state.

## Anti-Evidence

Both exports also retain exact rational companions (`details.price_r` for operations and `price_n`/`price_d` for offers), so exact reconciliation remains possible for consumers that ignore the scalar columns. But the ETL still publishes contradictory human-facing numeric price fields for the same ledger fact, which can silently corrupt joins and analytics that use those primary numeric columns.
