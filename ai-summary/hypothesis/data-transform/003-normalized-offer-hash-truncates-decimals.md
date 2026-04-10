# H003: Normalized offer IDs collide after 6-decimal formatting

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`DimOfferID` should change whenever the normalized offer state changes in a way the table claims to encode, including one-stroop differences in `BaseAmount` or materially different prices. Two distinct offer states should not hash to the same `DimOfferID` just because their decimal formatting rounded away the difference.

## Mechanism

`extractDimOffer()` hashes `fmt.Sprintf("%d/%f/%f", offer.OfferID, offer.Amount, offer.Price)`. Go's default `%f` prints only 6 decimal places, but `offer.Amount` is produced from stroops and carries 7-digit precision, and `offer.Price` can have even more precision than that. As a result, distinct offer states such as `262.8450327` and `262.8450326` stringify identically and produce the same FNV hash, so downstream normalized-offer rows can be deduplicated onto the wrong dimension key.

## Trigger

Export two states of the same offer ID whose amount changes by one stroop while the price stays constant. For example, the current test fixture amount `262.8450327` and an otherwise identical `262.8450326` both format to the same `%f` string, yielding the same `DimOfferID`.

## Target Code

- `internal/transform/offer.go:63-66,79-100` — offer price/amount are stored as `float64` values derived from on-chain quantities
- `internal/transform/offer_normalized.go:139-147` — `DimOfferID` hash key is built from `%f`-formatted amount and price
- `internal/transform/offer_normalized.go:159-167` — the resulting hash becomes the persisted normalized offer identifier
- `internal/transform/offer_normalized_test.go:106-113` — fixture already includes a 7-decimal amount and a price with more than 6 decimals

## Evidence

The test fixture in `offer_normalized_test.go:106-113` uses `BaseAmount: 262.8450327` and `Price: 0.5142373444404865`, both of which contain precision beyond `%f`'s default 6 decimals. `extractDimOffer()` then feeds those floats into `fmt.Sprintf("%d/%f/%f", ...)`, so the hashed identity string rounds them to `262.845033/0.514237`. A one-stroop change in amount therefore disappears before hashing.

## Anti-Evidence

If normalized offers were intentionally grouped at 6-decimal precision, the truncation would be expected. But the struct stores full `BaseAmount` and `Price` alongside the hash, so the current behavior creates a mismatch where the row claims one precise state while its deduplication key only encodes a rounded approximation.
