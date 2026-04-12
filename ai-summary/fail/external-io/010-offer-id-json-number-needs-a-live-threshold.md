# H010: Offer-detail `offer_id` values might need string encoding, but the live trigger is unproven

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: identifier precision / JSON contract drift
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If stellar-etl exports `offer_id` inside operation or effect `details`, the field should preserve the exact identifier and ideally follow Horizon's quoted-string JSON encoding so downstream systems never have to guess at 64-bit integer precision.

## Mechanism

`extractOperationDetails()` and `tradeDetails()` both write `offer_id` into generic `map[string]interface{}` values as raw integers, while Horizon tags the corresponding fields with `json:",string"`. That looks like the same kind of precision / contract bug as the muxed-ID paths.

## Trigger

1. Export a ledger containing a manage-offer operation or trade effect whose `offer_id` exceeds the IEEE-754 safe integer boundary (`> 9007199254740992`).
2. Observe whether the JSON output emits a bare numeric `offer_id` that downstream generic JSON decoders round.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:701-727` — writes manage-buy / manage-sell `offer_id` into `details`
- `internal/transform/effects.go:tradeDetails:1229-1241` — writes trade-effect `offer_id` into generic maps

## Evidence

The repo stores `offer_id` as a raw integer in both live detail builders, and the upstream Horizon operation/effect resource types declare `offer_id` with `json:",string"`. On code shape alone, this looked like a plausible precision bug.

## Anti-Evidence

Unlike muxed IDs and sequence numbers, offer IDs are allocated by core rather than directly supplied by a transaction author. I did not find a concrete, present-day ledger range or code-level source showing that legitimate live `offer_id` values can already cross the JSON-safe integer threshold, so the impact remains speculative.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The encoding concern is real in principle, but I could not establish the required concrete trigger on today's live system. Without evidence that legitimate `offer_id` values can presently exceed `2^53`, this remains a future-only speculation rather than a current data-integrity bug.

### Lesson Learned

For JSON-number exactness findings in this repo, XDR width alone is not enough. The field also needs a clearly reachable live range crossing the IEEE-754 precision boundary, or it belongs in fail rather than hypothesis.
