# H004: `offers.amount` rounds distinct large offer sizes together

**Date**: 2026-04-14
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Offer-state export should preserve the exact remaining offer amount represented by `OfferEntry.Amount`. Two valid offers whose remaining amounts differ by 1 stroop should export different `offers.amount` values.

## Mechanism

`TransformOffer()` takes the exact `Int64` `OfferEntry.Amount` and converts it to `float64` with `utils.ConvertStroopValueToReal()` before storing it in `OfferOutput.Amount`. At sufficiently large magnitudes, adjacent offer sizes collapse to the same plausible decimal value, so downstream order-book reconstruction sees a silently corrupted size even though the XDR source remained exact up to that final assignment.

## Trigger

Feed `TransformOffer()` two otherwise identical offer ledger entries whose `Amount` values are `90071992547409930` and `90071992547409931` stroops. Compare the resulting `OfferOutput.Amount` values after JSON export: both should serialize to the same rounded number despite the one-stroop difference on-chain.

## Target Code

- `internal/transform/offer.go:44-47` — reads exact `OfferEntry.Amount`
- `internal/transform/offer.go:79-101` — writes `OfferOutput.Amount` via `ConvertStroopValueToReal`
- `internal/transform/schema.go:260-282` — `OfferOutput.Amount` is the exported state column
- `internal/utils/main.go:84-87` — stroop conversion returns `float64`
- `.../xdr/xdr_generated.go:5427-5433` — `OfferEntry.Amount` is exact XDR `Int64`

## Evidence

The transform path keeps `OfferEntry.Amount` as an integer until the final row assembly, so the corruption is introduced entirely by the ETL's own type choice. This field is especially important because sibling normalized-offer logic already depends on `OfferOutput.Amount`, so the rounded value can contaminate both raw offer exports and derived order-book tables.

## Anti-Evidence

The bug requires unusually large outstanding offers; ordinary sizes still export distinctly. The existing offer-detail rounding finding only covers operation-detail maps, not the persisted offer-state table targeted here.
