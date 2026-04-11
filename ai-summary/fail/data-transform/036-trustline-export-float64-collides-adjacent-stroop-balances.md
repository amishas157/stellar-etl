# H003: Trustline export rounds adjacent stroop balances to the same float64

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Two trustline states that differ by one stroop should export different `balance`, `buying_liabilities`, or `selling_liabilities` values. A trustline with `10735946655003267` stroops and one with `10735946655003268` stroops should not collapse to the same decimal output.

## Mechanism

`TransformTrustline()` sends trustline balance and liabilities through `ConvertStroopValueToReal()`, and `TrustlineOutput` stores those results as `float64`. At high magnitudes the float cannot preserve single-stroop deltas, so distinct trustline balances become numerically identical in exported rows even though the underlying XDR `Int64` values differ.

## Trigger

Process a trustline ledger entry whose `Balance` or liabilities are at or above `10735946655003267` stroops. Export two states differing by one stroop and observe that both rows emit the same floating-point value instead of neighboring 7-decimal amounts.

## Target Code

- `internal/transform/trustline.go:TransformTrustline:59-88` - trustline balance and liabilities are converted to `float64`.
- `internal/transform/schema.go:TrustlineOutput:237-258` - those exported monetary fields are typed as `float64`.
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` - conversion endpoint that loses single-stroop precision at high values.

## Evidence

The trustline transform uses the same float conversion path for three monetary columns, so the same adjacent-stroop collision appears here as in other stroop-denominated outputs. The target table has no raw-stroop companion column for `balance` or liabilities, so the lost unit cannot be recovered from the exported trustline row.

## Anti-Evidence

Trustline balances near the collision threshold are rarer than ordinary balances, so the issue may not appear often in typical ledgers. Reviewers may also view this as a shared schema problem across several float-based tables rather than a trustline-specific defect.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantively equivalent to 034-account-export-float64-collides-adjacent-stroop-balances
**Failed At**: reviewer

### Trace Summary

Traced `TransformTrustline()` in `internal/transform/trustline.go:68-88` which converts Balance (line 75), BuyingLiabilities (line 78), and SellingLiabilities (line 79) through `utils.ConvertStroopValueToReal()`. This helper at `internal/utils/main.go:85-87` uses `big.NewRat(int64(input), int64(10000000)).Float64()`, which produces the nearest representable float64 — the best possible float64 conversion. The `TrustlineOutput` schema defines these fields as `float64` with no companion raw stroop field. This is the exact same pattern already rejected in 034-account-export-float64-collides-adjacent-stroop-balances.

### Code Paths Examined

- `internal/transform/trustline.go:TransformTrustline:75` — `Balance: utils.ConvertStroopValueToReal(trustEntry.Balance)` uses the shared helper
- `internal/transform/trustline.go:TransformTrustline:78-79` — `BuyingLiabilities` and `SellingLiabilities` use the same shared helper
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — `big.NewRat(...).Float64()` produces the nearest float64 (best possible conversion)
- `internal/transform/schema.go:TrustlineOutput` — Balance, BuyingLiabilities, SellingLiabilities are `float64` with no raw stroop companion

### Why It Failed

This is substantively equivalent to the already-rejected hypothesis 034-account-export-float64-collides-adjacent-stroop-balances. The trustline transform uses the identical conversion path (`ConvertStroopValueToReal` → `big.NewRat().Float64()`) and the identical schema design (`float64` fields). The precision loss is an inherent property of the float64 schema type, not a coding bug. Unlike confirmed findings 016 (claimable-balance inline conversion bypassing the shared helper) and 017 (LP-deposit discarding an exact string via ParseFloat), the trustline transform correctly uses the established best-practice helper — there is no "better path available but not used." This is a schema design choice aligned with BigQuery, not a fixable bug.

### Lesson Learned

Hypotheses about float64 precision loss in different entity transforms (account, trustline, offer, etc.) that all use the shared `ConvertStroopValueToReal()` helper are the same root finding — an inherent schema limitation. Only cases where a better conversion path exists but is bypassed (inline math, ParseFloat on exact strings) qualify as bugs.
