# H001: Claimable-balance `asset_amount` uses the lossy inline float path instead of the shared stroop helper

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`claimable_balances.asset_amount` should use the same best-available stroop-to-float conversion as the rest of the transform package so that large balances preserve every distinguishable 1-stroop step that fits in the JSON float schema. For a claimable balance amount of `9007199254740993` stroops, the exported value should be `900719925.4740993`, matching `utils.ConvertStroopValueToReal(xdr.Int64(amount))`.

## Mechanism

`TransformClaimableBalance()` computes `asset_amount` with raw `float64(outputAmount) / 1.0e7` instead of the shared `ConvertStroopValueToReal()` helper. In Go, those two paths diverge above the IEEE-754 integer-exactness boundary: `9007199254740993` becomes `900719925.4740992` through the inline division but `900719925.4740993` through `big.NewRat(...).Float64()`, so the claimable-balance export silently loses a stroop that sibling balance exports would still preserve.

## Trigger

Process any claimable-balance ledger entry whose `ClaimableBalanceEntry.Amount` exceeds `2^53+1` stroops, for example `9007199254740993`.

1. `TransformClaimableBalance()` will export `asset_amount = 900719925.4740992`.
2. The same raw amount passed through `utils.ConvertStroopValueToReal()` would export `900719925.4740993`.
3. Downstream consumers joining claimable balances against other balance-style tables will see a 1-stroop mismatch for the same on-chain quantity.

## Target Code

- `internal/transform/claimable_balance.go:48-76` — converts `ClaimableBalanceEntry.Amount` with inline `float64(...)/1.0e7`
- `internal/utils/main.go:84-88` — shared stroop helper uses `big.NewRat(...).Float64()`
- `internal/transform/claimable_balance_test.go:107-131` — current tests only cover a small amount (`9990000000`), so the precision boundary is untested

## Evidence

The package already centralized stroop conversion in `utils.ConvertStroopValueToReal()` and uses that helper for account, trustline, offer, trade, pool, and most operation amount fields. A direct Go comparison shows the claimable-balance code path is observably worse: for `9007199254740993`, inline division yields `900719925.4740992` while the helper yields `900719925.4740993`.

## Anti-Evidence

The output schema is still `float64`, so very large values will eventually collide even on the helper path. This hypothesis is narrower: it targets the avoidable extra rounding introduced by bypassing the helper, not the broader float-schema limitation already known elsewhere in the codebase.

---

## Review

**Verdict**: NOT_VIABLE — duplicate of `ai-summary/success/data-transform/016-claimable-balance-inline-float64-rounding.md`
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of success/data-transform/016-claimable-balance-inline-float64-rounding
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same bug as the already-published finding `016-claimable-balance-inline-float64-rounding` in `ai-summary/success/data-transform/`. Both identify that `TransformClaimableBalance()` uses inline `float64(amount) / 1.0e7` instead of the shared `utils.ConvertStroopValueToReal()` helper, producing `900719925.4740992` instead of `900719925.4740993` for amounts exceeding 2^53 stroops. The published finding already includes a full PoC, adversarial review, and suggested fix.

### Code Paths Examined

- `ai-summary/success/data-transform/016-claimable-balance-inline-float64-rounding.md.gh-published` — confirmed identical mechanism, trigger, target code, and impact

### Why It Failed

This is an exact duplicate of an already-confirmed and published finding. The same root cause (inline float64 division bypassing the shared stroop helper), the same trigger (large `ClaimableBalanceEntry.Amount`), and the same observable impact (1-stroop rounding error in exported JSON) were already investigated, PoC-verified, and published as success finding 016.

### Lesson Learned

Always check `ai-summary/success/{SUBSYSTEM}/` for existing published findings before submitting a hypothesis. This exact bug was already confirmed and published under the same subsystem.
