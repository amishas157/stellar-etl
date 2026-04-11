# H002: Account export rounds adjacent stroop balances to the same float64

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Two account ledger entries whose balances differ by one stroop should export distinct decimal balances. For example, balances of `10735946655003267` and `10735946655003268` stroops should become `1073594665.5003267` and `1073594665.5003268`, and the same one-stroop distinction should hold for buying/selling liabilities.

## Mechanism

`TransformAccount()` converts account balance and liabilities to `float64`, and `AccountOutput` stores those monetary fields as `float64`. `ConvertStroopValueToReal()` ultimately returns a binary float, so adjacent high stroop values collide: both `10735946655003267` and `10735946655003268` export as `1073594665.5003268`, silently erasing one stroop of balance precision in the account table.

## Trigger

Process an account ledger entry whose `Balance`, `Liabilities.Buying`, or `Liabilities.Selling` is at or above `10735946655003267` stroops. Compare two otherwise identical entries that differ by one stroop: the JSON export should differ, but both rows serialize to the same `float64` value.

## Target Code

- `internal/transform/account.go:TransformAccount:29-110` - balance and liabilities are converted with `utils.ConvertStroopValueToReal()`.
- `internal/transform/schema.go:AccountOutput:97-121` - exported balance fields are typed as `float64`.
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` - exact stroop integers are reduced to `float64`.

## Evidence

The account transform emits three monetary fields (`balance`, `buying_liabilities`, `selling_liabilities`) through the same `float64` conversion path. Reproducing that conversion with adjacent stroop values such as `10735946655003267` and `10735946655003268` yields the same exported decimal `1073594665.5003268`.

## Anti-Evidence

This only manifests for sufficiently large balances and liabilities; smaller account values remain distinguishable. The JSON schema is already defined as `float64`, so reviewers may decide this is an inherited schema constraint rather than a bug unless exact-stroop fidelity is required for the table.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of 034-account-export-float64-collides-adjacent-stroop-balances.md
**Failed At**: reviewer

### Trace Summary

This hypothesis is an exact duplicate of the previously reviewed and rejected hypothesis at `ai-summary/fail/data-transform/034-account-export-float64-collides-adjacent-stroop-balances.md`. That prior review already traced the full code path and concluded the finding is NOT_VIABLE.

### Code Paths Examined

- `ai-summary/fail/data-transform/034-account-export-float64-collides-adjacent-stroop-balances.md` — identical hypothesis with complete NOT_VIABLE review already on file

### Why It Failed

Duplicate of 034. The prior review established that the account transform correctly uses `ConvertStroopValueToReal()` with `big.NewRat().Float64()`, which is the best possible float64 conversion. Unlike confirmed findings 016 (claimable balance) and 017 (LP deposit) where a worse conversion path existed, the account transform already uses the project's established correct conversion helper. The precision loss is an inherent property of the float64 schema type, not a coding bug.

### Lesson Learned

Check existing fail entries for the same subsystem before re-submitting; this exact hypothesis was already fully reviewed at 034.
