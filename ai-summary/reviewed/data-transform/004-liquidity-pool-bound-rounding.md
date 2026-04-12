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

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the `extractOperationDetails()` function for both `LiquidityPoolDeposit` (lines 957-1019) and `LiquidityPoolWithdraw` (lines 1021-1062) branches. Confirmed that `reserve_a_max_amount` (line 990), `reserve_b_max_amount` (line 1001), `reserve_a_min_amount` (line 1051), and `reserve_b_min_amount` (line 1058) are all converted via `utils.ConvertStroopValueToReal()`, which calls `big.NewRat(int64(input), 10000000).Float64()` — returning the nearest float64 representation. The sibling formatter at lines 1631-1634 and 1676-1679 uses exact `amount.String()` for the same logical values, confirming the lossy path is unnecessary.

### Code Paths Examined

- `internal/transform/operation.go:990` — `details["reserve_a_max_amount"] = utils.ConvertStroopValueToReal(op.MaxAmountA)` — lossy float64 conversion of deposit max bound
- `internal/transform/operation.go:1001` — `details["reserve_b_max_amount"] = utils.ConvertStroopValueToReal(op.MaxAmountB)` — lossy float64 conversion of deposit max bound
- `internal/transform/operation.go:1051` — `details["reserve_a_min_amount"] = utils.ConvertStroopValueToReal(op.MinAmountA)` — lossy float64 conversion of withdraw min bound
- `internal/transform/operation.go:1058` — `details["reserve_b_min_amount"] = utils.ConvertStroopValueToReal(op.MinAmountB)` — lossy float64 conversion of withdraw min bound
- `internal/utils/main.go:ConvertStroopValueToReal` — `big.NewRat(int64(input), 10000000).Float64()` returns nearest float64
- `internal/transform/operation.go:1631-1634` — sibling formatter uses `amount.String(op.MaxAmountA)` for exact deposit bounds
- `internal/transform/operation.go:1676-1679` — sibling formatter uses `amount.String(op.MinAmountA)` for exact withdraw bounds

### Findings

The four request-side reserve bound fields (`reserve_a_max_amount`, `reserve_b_max_amount`, `reserve_a_min_amount`, `reserve_b_min_amount`) are distinct from the realized delta fields covered by success/017. These fields represent the depositor's maximum willingness or the withdrawer's minimum acceptable amounts — monetary constraint values that downstream systems use to understand transaction intent. The `ConvertStroopValueToReal()` conversion loses precision for values above ~9007199254 XLM (2^53 stroops), causing adjacent 1-stroop-different bounds to collapse to the same float64. The sibling formatter in the same file proves exact decimal output was available and intended.

Additionally, the withdraw branch has three more fields using the same lossy conversion: `reserve_a_withdraw_amount` (line 1052), `reserve_b_withdraw_amount` (line 1059), and `shares` (line 1061) — these are also affected but the hypothesis correctly focuses on the request-side bounds as the primary novel finding.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or a new `data_integrity_poc_test.go` in the same package)
- **Setup**: Construct two `LiquidityPoolDepositOp` operations with `MaxAmountA` values of `90071992547409930` and `90071992547409931` stroops. Similarly construct two `LiquidityPoolWithdrawOp` operations with `MinAmountA` values differing by 1 stroop at the same magnitude. Wrap in valid `ingest.LedgerTransaction` with successful results and LP ledger changes.
- **Steps**: Call `extractOperationDetails()` or `TransformOperation()` for each pair.
- **Assertion**: Assert that `details["reserve_a_max_amount"]` (deposit) and `details["reserve_a_min_amount"]` (withdraw) produce the same float64 for two inputs that differ by 1 stroop, demonstrating the precision loss. Compare against `amount.String()` output to show the exact values are distinct.
