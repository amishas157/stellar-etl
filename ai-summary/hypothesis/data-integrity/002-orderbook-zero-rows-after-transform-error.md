# H002: Orderbook parser emits zero-valued rows after a failed offer transform

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `TransformOfferNormalized()` fails for an offer change, that offer should be skipped entirely: no `dim_markets`, `dim_accounts`, `dim_offers`, or `fact_offer_events` rows should be emitted for that failed input. Failed transformations should reduce coverage, not fabricate placeholder records.

## Mechanism

`parseOrderbook()` preallocates `allConverted` and launches `convertOffer()` goroutines to fill each index. When a transform fails, `convertOffer()` only logs the error and leaves that slot at its Go zero value; `parseOrderbook()` then iterates every slot unconditionally and marshals the zero-valued `Market`, `Account`, `Offer`, and `Event`. A single rejected offer can therefore create bogus rows with ID `0`, empty strings, and `ledger_id=0`, which look structurally valid enough to poison downstream orderbook tables.

## Trigger

Feed `parseOrderbook()` a batch containing any offer change that makes `TransformOfferNormalized()` return an error, such as a deleted offer (rejected at `TransformOfferNormalized()` before normalization). The batch output will contain one synthetic zero row for the failed slot.

## Target Code

- `internal/input/orderbooks.go:37-45` — `convertOffer()` logs transform errors but does not mark the slot invalid
- `internal/input/orderbooks.go:62-117` — `parseOrderbook()` marshals every `allConverted` entry, including untouched zero values
- `internal/transform/offer_normalized.go:16-26` — deleted offers are a concrete path that returns an error instead of a normalized record

## Evidence

`allConverted := make([]transform.NormalizedOfferOutput, len(orderbook))` initializes every slot to zero values. On error, `convertOffer()` skips `allConvertedOffers[index] = transformed`, so the later loop sees a default `NormalizedOfferOutput{}` and still appends its marshaled children unless a later marshal itself fails.

## Anti-Evidence

The seen-hash maps limit the damage after the first zero-valued market/account/offer is inserted, so repeated failures will tend to reuse the same bogus IDs rather than creating infinitely many variants. That still leaves at least one bad dimension row plus zero-valued events for each failed slot.
