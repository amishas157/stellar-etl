# H005: Liquidity-pool withdraw details round large reserve and share amounts

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A successful `liquidity_pool_withdraw` row should preserve the exact 7-decimal values for `reserve_a_min_amount`, `reserve_b_min_amount`, `reserve_a_withdraw_amount`, `reserve_b_withdraw_amount`, and `shares`. A withdrawal of `90071992547409931` stroops should export the exact decimal `9007199254.7409931`, and 1-stroop differences should remain visible.

## Mechanism

In `extractOperationDetails()`, all five of these withdraw fields are emitted through `utils.ConvertStroopValueToReal()`. The sibling `transactionOperationWrapper.Details()` path for the same operation already exports exact `amount.String(...)` values inside `reserves_min`, `reserves_received`, and `shares`, so the lossy float64 encoding is not required by the operation-details contract and can silently collapse distinct reserve/share deltas.

## Trigger

Process a successful `liquidity_pool_withdraw` whose min amounts, received reserves, or withdrawn share amount are large, for example:

1. `LiquidityPoolWithdrawOp.Amount = 90071992547409931`, or
2. a pool delta where `receivedA` / `receivedB = 90071992547409931`

Compare the exported detail fields against the exact `amount.String(...)` values or against a second withdrawal that differs by 1 stroop; the float fields will round together.

## Target Code

- `internal/transform/operation.go:1021-1061` — withdraw branch converts reserve mins, received amounts, and `shares` through `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1650-1684` — sibling wrapper emits exact strings for `reserves_min`, `reserves_received`, and `shares`
- `internal/transform/schema.go:136-150` — operation details are stored in an untyped map, so exact strings are representable

## Evidence

This is the withdraw-side analog of the already-confirmed LP-deposit precision bug, but with a different branch and a different exported key set. The same file already contains an exact-string representation for withdraw details, which makes the float64 conversion a branch-local data-loss decision rather than an unavoidable limitation.

## Anti-Evidence

There is already a rejected investigation about failed LP operations defaulting reserve assets to `"native"`; this hypothesis is different because it targets successful large-value withdrawals and exact-value loss, not failed-operation placeholder semantics. Current fixtures use tiny pool deltas, so they do not exercise the precision boundary.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (but substantially equivalent to rejected pattern 034-038)
**Failed At**: reviewer

### Trace Summary

The withdraw branch at `extractOperationDetails()` lines 1021-1062 converts all five financial fields directly from `xdr.Int64` to float64 via `utils.ConvertStroopValueToReal()`, which uses `big.NewRat(int64(input), int64(10000000)).Float64()` — the best possible float64 conversion available in Go. Unlike the sibling deposit branch (confirmed bug 017), which computed exact strings with `amount.String()` and then unnecessarily round-tripped them through `strconv.ParseFloat()`, the withdraw branch never computes an exact string and never discards one. The cited alternative path `transactionOperationWrapper.Details()` (line 1364) is dead code — never called anywhere in the codebase.

### Code Paths Examined

- `internal/transform/operation.go:1021-1062` — withdraw branch uses `utils.ConvertStroopValueToReal()` directly for all five fields (`reserve_a_min_amount`, `reserve_a_withdraw_amount`, `reserve_b_min_amount`, `reserve_b_withdraw_amount`, `shares`); no exact string is computed or discarded
- `internal/utils/main.go:85-88` — `ConvertStroopValueToReal()` uses `big.NewRat(int64, 1e7).Float64()`, the highest-precision float64 conversion path available
- `internal/transform/operation.go:957-1019` — deposit branch, by contrast, calls `strconv.ParseFloat(amount.String(...))` which computes an exact string then discards it (this was the confirmed bug 017)
- `internal/transform/operation.go:1364` — `transactionOperationWrapper.Details()` is defined but never called by any code in the repository (confirmed via grep across all .go files — zero callers)
- `internal/transform/operation.go:1650-1684` — the withdraw formatter inside dead-code `Details()` uses `amount.String()` for exact output, but this path is unreachable

### Why It Failed

This hypothesis falls into the same category as previously rejected fail entries 034-038 and 039: the code uses `ConvertStroopValueToReal()` → `big.NewRat().Float64()`, which is the best possible float64 conversion. The precision loss is an inherent property of the float64 schema type, not a coding error. The critical distinction from the VIABLE deposit finding (017) is that the deposit branch **computed an exact string with `amount.String()` and then discarded it** via `ParseFloat` — a concrete coding mistake where a better value was available and thrown away. The withdraw branch does no such thing; it goes directly from `xdr.Int64` to the optimal float64 conversion without ever producing an exact intermediate value. The cited `transactionOperationWrapper.Details()` is dead code (zero callers in the entire codebase) and per meta-pattern #10, dead code cannot serve as evidence of "a better path available but not used."

### Lesson Learned

The deposit and withdraw LP branches use fundamentally different conversion strategies despite being sibling operations. The deposit branch was VIABLE (017) because it used `strconv.ParseFloat(amount.String(...))` — computing an exact string and then discarding it. The withdraw branch is NOT_VIABLE because it uses `ConvertStroopValueToReal()` directly — the best possible float64 conversion with no exact intermediate being thrown away. Always verify the specific conversion mechanism in each branch rather than assuming sibling operations share the same pattern.
