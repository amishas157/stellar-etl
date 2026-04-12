# H004: Liquidity-pool request bounds round large reserve limits in operation details

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `liquidity_pool_deposit` and `liquidity_pool_withdraw`, the request-side reserve bounds in `history_operations.details` should preserve the exact on-chain stroop values. Fields such as `reserve_a_max_amount`, `reserve_b_max_amount`, `reserve_a_min_amount`, and `reserve_b_min_amount` should remain distinct when the underlying XDR values differ by 1 stroop.

## Mechanism

The live deposit and withdraw branches convert these bound fields through `utils.ConvertStroopValueToReal()`, so large values lose low-order stroop precision before export. The same file's alternate formatter already emits the corresponding reserve bounds with exact `amount.String(...)` values inside `reserves_max` / `reserves_min`, which shows this decoded JSON surface can carry exact decimal strings without changing the schema type of `OperationDetails` itself.

## Trigger

1. Build a successful `liquidity_pool_deposit` with `MaxAmountA` or `MaxAmountB` equal to `90071992547409930` stroops and a second one with `90071992547409931`.
2. Build a successful `liquidity_pool_withdraw` with `MinAmountA` or `MinAmountB` differing by 1 stroop at the same magnitude.
3. Run them through `TransformOperation()` and compare the exported reserve bound fields.

## Target Code

- `internal/transform/operation.go:957-1013` — deposit detail export converts `reserve_a_max_amount` / `reserve_b_max_amount` with `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1021-1060` — withdraw detail export converts `reserve_a_min_amount` / `reserve_b_min_amount` with `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1603-1684` — sibling formatter keeps exact bound values via `amount.String(...)`

## Evidence

The live output uses exact decimal strings for some adjacent liquidity-pool detail fields (`shares` in withdraw, `destination_min` elsewhere in the file) but still writes the request-side reserve bounds as `float64`. The alternate formatter preserves those same bounds exactly in the same source file, so large bound values can silently collapse only because the active branch chooses the lossy path.

## Anti-Evidence

The realized reserve deltas and share outputs are separate fields, and the envelope XDR can still be decoded elsewhere. But downstream users reading `history_operations.details` as the decoded request payload will see incorrect reserve constraints for large LP operations.
