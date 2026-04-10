# H004: `DimOfferID` hashing truncates 7-digit offer precision and collapses distinct orderbook states

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Two states of the same Horizon offer should receive different `dim_offer_id` values whenever either the remaining amount or effective price changes. In particular, offer amounts that differ by one stroop (`0.0000001`) must remain distinguishable, because Stellar orderbook amounts are tracked at 7-digit precision.

## Mechanism

`TransformOffer()` converts offer amounts to `float64` at 7-digit stroop precision, but `extractDimOffer()` hashes `fmt.Sprintf("%d/%f/%f", offer.OfferID, offer.Amount, offer.Price)`. The default `%f` formatter keeps only 6 digits after the decimal point, so distinct states like `1.0000001` and `1.0000002` serialize to the same string and therefore the same FNV hash. `OrderbookParser` then deduplicates `dim_offers` by `DimOfferID`, silently dropping one of the distinct amount/price states while still emitting fact rows that point at the collapsed identifier.

## Trigger

Process two ledger changes for the same `offer_id` where the remaining amount or price differs only at the 7th decimal place, such as an offer moving from `1.0000001` XLM to `1.0000002` XLM. The normalized orderbook export will reuse the same `dim_offer_id` and keep only one `dim_offers` row even though the actual financial state changed.

## Target Code

- `internal/transform/offer.go:44-90` — converts on-chain stroops to `OfferOutput.Amount` with 7-digit precision
- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal()` preserves the 1e-7 unit before formatting
- `internal/transform/offer_normalized.go:139-147` — hashes `OfferID`, `Amount`, and `Price` via `%f`
- `internal/input/orderbooks.go:97-107` — deduplicates `dim_offers` solely by `converted.Offer.DimOfferID`

## Evidence

The collision mechanism is explicit: `%f` rounds to six fractional digits unless precision is specified. Because `OfferOutput.Amount` comes from stroops divided by `1e7`, one-stroop deltas live exactly in the digit that gets truncated before hashing.

## Anti-Evidence

If offer updates change by more than `0.000001` or alter the integer part, the hash input remains distinct. The bug only manifests for fine-grained amount/price changes, but those are legitimate and common on a 7-decimal Stellar orderbook.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-integrity/001-dim-offer-id-collision.md.gh-published`
**Failed At**: reviewer

### Trace Summary

The hypothesis describes exactly the same bug as the already-confirmed finding in `success/data-integrity/001-dim-offer-id-collision.md.gh-published`. Both identify `extractDimOffer()` at `offer_normalized.go:140` using `fmt.Sprintf("%d/%f/%f", ...)` where default `%f` truncates to 6 decimal places, losing the 7th stroop digit. Both trace the same deduplication path through `OrderbookParser` and identify the same impact on `dim_offers` rows.

### Code Paths Examined

- `internal/transform/offer_normalized.go:139-147` — `extractDimOffer` uses `fmt.Sprintf("%d/%f/%f", offer.OfferID, offer.Amount, offer.Price)` with default `%f` (6 decimal places)
- `internal/transform/offer.go:90` — `Amount` set via `utils.ConvertStroopValueToReal(outputAmount)` producing 7-digit precision float64
- `internal/transform/schema.go` — `OfferOutput.Amount` is `float64`, `OfferOutput.Price` is `float64`

### Why It Failed

This is a duplicate of the already-confirmed and published finding `001-dim-offer-id-collision.md.gh-published` in `success/data-integrity/`. The confirmed finding covers the identical root cause (`%f` truncation in `extractDimOffer`), the identical code path (`offer_normalized.go` → `orderbooks.go` dedup), and the identical impact (DimOfferID hash collisions for one-stroop deltas). The hypothesis adds no new information beyond what was already discovered and validated.

### Lesson Learned

Before generating hypotheses about `%f` formatting precision in hash functions, check whether the same formatting/hashing path has already been flagged as a confirmed finding.
