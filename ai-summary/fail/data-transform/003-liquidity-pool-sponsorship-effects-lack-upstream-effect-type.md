# H003: Sponsored liquidity-pool entries should emit sponsorship effects

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the effects export were supposed to track standalone liquidity-pool sponsorship lifecycle changes, then sponsored liquidity-pool ledger-entry updates should emit dedicated `liquidity_pool_sponsorship_created`, `..._updated`, or `..._removed` effects with the pool identifier in `details`.

## Mechanism

This looked plausible because sponsored-reserves support covers liquidity pools, while `sponsoringEffectsTable` only maps account, trustline, data, claimable-balance, and signer sponsorship effects. But the missing local mapping is not a live ETL bug: the local effect enum set in `schema.go` mirrors the current Horizon-style sponsorship effect taxonomy, and that taxonomy does not define standalone liquidity-pool sponsorship effect types. Without an upstream effect contract to preserve, the omission is a product-surface choice rather than local data corruption.

## Trigger

1. Inspect `sponsoringEffectsTable` and note that `LedgerEntryTypeLiquidityPool` is absent.
2. Inspect the effect enum/name tables and note that no `liquidity_pool_sponsorship_*` effect types exist.
3. Compare that local effect set against the current published effect-type catalog, which likewise has sponsorship effects for account/trustline/data/claimable-balance/signer but not for standalone liquidity-pool sponsorship rows.

## Target Code

- `internal/transform/effects.go:200-226` — sponsorship effect table omits `LedgerEntryTypeLiquidityPool`
- `internal/transform/effects.go:290-362` — sponsorship effect generation can only emit mapped effect categories
- `internal/transform/schema.go:404-474` — effect enum/name tables define no `liquidity_pool_sponsorship_*` effect types

## Evidence

The effect generator has no enum values or names for liquidity-pool sponsorship effects, not just a missing case in one switch. The surrounding effect taxonomy aligns with the current published Horizon-style sponsorship effect families, which stop at account, trustline, data, claimable balance, and signer sponsorship. That made the omission look more like an upstream-facing schema boundary than a local transform bug.

## Anti-Evidence

Sponsored-reserve documentation does make liquidity-pool sponsorship sound like a candidate for separate tracking, so the absence is not obvious from the protocol feature alone. The local `effects.go` comment claiming liquidity pools cannot be sponsored is also misleading and made this path worth tracing.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

There is no current upstream effect type contract for standalone liquidity-pool sponsorship rows. The local omission therefore does not drop a defined exported value; it reflects the current effect taxonomy boundary.

### Lesson Learned

For effects findings, missing switch cases are only actionable when the corresponding effect type already exists in the exported schema or upstream API contract. Check the effect enum/name catalog before assuming a newly sponsorable ledger entry type should produce a brand-new family of effect rows.
