# H005: Trade effects over-emit offer lifecycle rows for every claim atom

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: trade-related effect rows could claim impossible simultaneous offer states
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a non-liquidity-pool trade claim, the effect stream should emit only the offer-lifecycle effect that matches the actual post-claim offer state. A partially filled offer should produce `offer_updated`, a fully consumed offer should produce `offer_removed`, and a path-payment-created synthetic offer should produce `offer_created` only when that specific state transition actually happened.

## Mechanism

`addClaimTradeEffects()` loops over `trade`, `offer_updated`, `offer_removed`, and `offer_created` and appends buyer and seller effects for each entry. On first read this looks like a strong integrity bug because a single claim atom appears to generate mutually exclusive offer lifecycle effects simultaneously, creating plausible but contradictory effect rows.

## Trigger

1. Export `history_effects` for any trade-producing non-liquidity-pool operation.
2. Inspect the effect rows generated from a single claim atom.
3. The local code appears to emit `trade`, `offer_updated`, and `offer_removed` together, plus `offer_created` for non-path-payment flows.

## Target Code

- `internal/transform/effects.go:985-1013` — local `addClaimTradeEffects()` iterates all four effect types
- `internal/transform/effects_test.go:560-660` — local test expectations include the same multi-effect fanout
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/effects/effects.go:1112-1135` — upstream Horizon processor uses the identical loop
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/effects/effects_test.go:614-674` — upstream tests assert the same offer-updated/removed fanout

## Evidence

The local implementation unconditionally iterates `EffectTrade`, `EffectOfferUpdated`, `EffectOfferRemoved`, and `EffectOfferCreated`, which superficially suggests that mutually exclusive offer states are all exported for the same claim. The local test suite also expects that fanout, so the behavior is definitely live.

## Anti-Evidence

The exact same code shape exists in the upstream `stellar/go` effects processor, and upstream tests explicitly assert the same trade-effect mix. That strongly suggests this is an inherited Horizon effect contract rather than a local transform divergence.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected over-fanout is not a stellar-etl-specific corruption path: the local code mirrors the upstream Horizon effects processor byte-for-byte at this branch, and both repos' tests assert the same multi-effect output. This is an ecosystem effect-model convention, not a local transform regression.

### Lesson Learned

For suspicious effect fanout, compare the local branch against the upstream `stellar/go` effects processor before filing a transform bug. If the same effect mix is implemented and tested upstream, the issue belongs to the shared Horizon contract unless the ETL adds extra corruption on top.
