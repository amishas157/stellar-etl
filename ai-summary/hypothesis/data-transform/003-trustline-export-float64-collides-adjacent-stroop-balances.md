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
