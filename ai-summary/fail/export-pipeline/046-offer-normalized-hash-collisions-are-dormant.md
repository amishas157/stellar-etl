# H046: `DimOfferID` hash collisions from `%f` formatting are dormant in the current export pipeline

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the normalized-orderbook export path were user-facing, `DimOfferID` should distinguish offers whose amount or price differ by less than the default six decimal places of `fmt.Sprintf("%f")`. Two distinct offers should not hash to the same identifier just because their decimal rendering was truncated before hashing.

## Mechanism

`extractDimOffer()` builds the hash input with `fmt.Sprintf("%d/%f/%f", offer.OfferID, offer.Amount, offer.Price)`, so differences beyond six fractional digits disappear before the FNV hash is computed. That would make distinct offers collide on `DimOfferID` whenever the path is active.

## Trigger

Construct two normalized offers with the same `OfferID` but amounts or prices that only differ beyond the sixth decimal place, then feed them through `TransformOfferNormalized()`. The hash preimage would be identical even though the underlying offer values differ.

## Target Code

- `internal/transform/offer_normalized.go:139-147` — `extractDimOffer()` hashes `%f`-formatted amount and price
- `internal/input/orderbooks.go:37-45` — orderbook parser is the only caller of `TransformOfferNormalized()`
- `internal/input/orderbooks.go:195-209` — orderbook state update helper
- `internal/input/orderbooks.go:212-240` — unexported streaming/receive helpers for orderbook batches

## Evidence

The `%f` format verb defaults to six digits after the decimal point, so hash preimages for values such as `0.1234564` and `0.1234565` collapse. Repository search shows the normalized-offer path is only referenced inside `internal/input/orderbooks.go`.

## Anti-Evidence

There is no active CLI export command that calls `StreamOrderbooks()`, `ReceiveParsedOrderbooks()`, or `TransformOfferNormalized()`, so the collision cannot currently corrupt any shipped export artifact. This is a real bug in dormant helper code, but not a live export-pipeline finding today.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The only reachable callers of the `%f`-hash path are internal orderbook helpers, and those helpers are not wired into any current CLI export command. Without a live command path, this remains dormant helper debt rather than an export-pipeline data-corruption bug.

### Lesson Learned

Pattern-7 precision bugs in hash or dedup code are only viable for this subsystem when the hashed dataset is actually exported. Before filing numeric-formatting issues, trace the path all the way to a real `cmd/export_*` entrypoint.
