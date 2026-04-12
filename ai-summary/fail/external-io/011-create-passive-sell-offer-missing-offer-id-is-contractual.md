# H011: `create_passive_sell_offer` looked like it dropped `offer_id`, but Horizon omits that field too

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: missing operation-detail field
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the canonical operation-details contract for the sell-offer family includes an `offer_id`, then successful `create_passive_sell_offer` rows should export that field too, just like `manage_buy_offer` and `manage_sell_offer`.

## Mechanism

`extractOperationDetails()` assigns `details["offer_id"]` for the manage-offer branches but not for `CreatePassiveSellOffer`, which initially looked like a sibling inconsistency that could hide a generated offer identifier from downstream analytics.

## Trigger

1. Export a successful `create_passive_sell_offer` operation.
2. Inspect its `details` payload for the presence or absence of `offer_id`.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:701-755` — manage-offer branches include `offer_id`, passive-sell branch does not

## Evidence

Within the repo, `ManageBuyOffer` and `ManageSellOffer` both set `details["offer_id"]`, while the immediately adjacent `CreatePassiveSellOffer` branch only exports amount / price / asset details.

## Anti-Evidence

The upstream Horizon operation schema defines `CreatePassiveSellOffer` as just `Offer` without an `OfferID` field, while `ManageBuyOffer` and `ManageSellOffer` add `OfferID` explicitly. The repo's older `transactionOperationWrapper.Details()` path also omits `offer_id` for passive-sell offers, which points to an intentional contract distinction rather than an accidental omission.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The sibling mismatch is only apparent. Horizon itself does not expose `offer_id` on `create_passive_sell_offer`, so stellar-etl's omission matches the upstream contract instead of dropping a field it should have exported.

### Lesson Learned

When one branch differs from siblings, check the protocol surface before assuming a copy-paste omission. Some offer-family operations intentionally expose different detail fields even when their payloads look closely related.
