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
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `TransformAccount()` in `internal/transform/account.go:86-90` which converts Balance, BuyingLiabilities, and SellingLiabilities through `utils.ConvertStroopValueToReal()`. This helper at `internal/utils/main.go:85-87` uses `big.NewRat(int64(input), int64(10000000)).Float64()`, which produces the **nearest representable float64** to the exact rational value — the best possible float64 conversion. The `AccountOutput` schema at `internal/transform/schema.go:97-121` defines these fields as `float64`, and no raw stroop field is preserved alongside them. The precision loss is an inherent property of the float64 type, not a conversion error.

### Code Paths Examined

- `internal/transform/account.go:TransformAccount:86-90` — confirmed Balance, BuyingLiabilities, SellingLiabilities all use `utils.ConvertStroopValueToReal()`
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — confirmed `big.NewRat(...).Float64()` produces the nearest float64 (best possible conversion)
- `internal/transform/schema.go:AccountOutput:97-121` — confirmed Balance, BuyingLiabilities, SellingLiabilities are `float64` with no companion raw stroop field
- `ai-summary/success/data-transform/016-claimable-balance-inline-float64-rounding.md.gh-published` — confirmed this finding explicitly distinguishes the schema design choice from the conversion bug: "the schema's use of float64 is a broader design choice, but this path is still wrong because it bypasses the shared stroop converter"

### Why It Failed

The hypothesis describes an inherent precision limitation of the `float64` schema type, not a coding bug. The account transform correctly uses `ConvertStroopValueToReal()`, which uses `big.NewRat().Float64()` to produce the nearest representable float64 — the best possible conversion within the float64 type. Unlike the confirmed findings (016-claimable-balance used an inline conversion that was *worse* than the helper; 017-LP-deposit discarded an exact string by parsing it back through ParseFloat), the account transform has no "better path available but not used." The precision loss is the inherent cost of the `float64` schema design choice aligned with BigQuery, and changing it would be a schema redesign, not a bug fix. The confirmed finding 016 explicitly acknowledges this distinction.

### Lesson Learned

When float64 precision loss is identified, distinguish between (a) conversion bugs where a better path exists but isn't used (VIABLE — as in findings 016 and 017) and (b) inherent schema design limitations where the code already uses the best possible conversion (NOT_VIABLE — a design choice, not a bug). The `ConvertStroopValueToReal()` helper using `big.Rat` is the project's established "correct" conversion path.
