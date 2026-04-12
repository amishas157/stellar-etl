# H002: Failed liquidity-pool withdrawals export zeroed reserve withdrawals

**Date**: 2026-04-12
**Subsystem**: data-input
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a `liquidity_pool_withdraw` operation fails, `export_operations` should not emit a `history_operations` row claiming any realized reserve withdrawal, because no pool reserves or pool-share balances changed on-chain. If the exporter keeps a failed-attempt row, it should not serialize `reserve_a_withdraw_amount` and `reserve_b_withdraw_amount` as executed zero-valued withdrawals.

## Mechanism

`GetOperations()` treats failed and successful transactions identically, so failed liquidity-pool withdrawals reach `TransformOperation`. The withdraw branch only computes actual reserve deltas when `transaction.Result.Successful()` is true; otherwise it leaves `receivedA` and `receivedB` at their zero values and still exports the row, producing a syntactically valid withdrawal record whose realized amounts are silently wrong.

## Trigger

1. Find a ledger range containing a failed `liquidity_pool_withdraw` transaction.
2. Run `stellar-etl export_operations --start-ledger <L> --end-ledger <L> ...`.
3. Inspect the exported row for that operation.
4. The correct output is no operation row at all, but the current path can emit a row with `details.reserve_a_withdraw_amount=0` and `details.reserve_b_withdraw_amount=0` even though the withdrawal never executed.

## Target Code

- `internal/input/operations.go:GetOperations:41-77` — does not filter out failed transactions before expanding operations
- `cmd/export_operations.go:25-54` — exports every transformed operation row
- `internal/transform/operation.go:69-98` — still returns an operation row when no per-operation results are available
- `internal/transform/operation.go:1021-1061` — `liquidity_pool_withdraw` keeps zero-value realized amounts unless the transaction succeeded

## Evidence

The withdraw branch initializes the realized reserve amounts to zero and only replaces them inside the `if transaction.Result.Successful()` block. The exported details always include request-side fields like `shares`, `reserve_a_min_amount`, and `reserve_b_min_amount`, so a failed withdrawal becomes a plausible row describing a zero-withdrawal event instead of a nonexistent operation. This conflicts with the ledger schema's distinction between successful operation count and all envelope operations.

## Anti-Evidence

The README's `export_operations` description is short and does not explicitly say "successful operations only." A reviewer may conclude the command intentionally exports attempted operations, but even under that reading the realized reserve-withdraw fields should not be serialized as actual zero withdrawals when the attempt never applied.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of 024-failed-liquidity-pool-deposit-exports-zero-amounts.md
**Failed At**: reviewer

### Trace Summary

This hypothesis is substantially equivalent to the already-investigated H001 (failed liquidity-pool deposit, filed as `024-failed-liquidity-pool-deposit-exports-zero-amounts.md`). That prior review explicitly traced the `LiquidityPoolWithdraw` code path at `operation.go:1037` and identified it as part of the same deliberate, codebase-wide design pattern applied across 6+ operation types. The withdraw branch at lines 1021-1061 uses the identical structure: zero-default `receivedA`/`receivedB`, populated only inside `if transaction.Result.Successful()`, with an explicit comment at line 1038 explaining the intent.

### Code Paths Examined

- `internal/transform/operation.go:1021-1061` — `LiquidityPoolWithdraw` branch uses the identical pattern to `LiquidityPoolDeposit`: zero defaults for `receivedA`/`receivedB`, only replaced on success. Comment at line 1038: "we will use the defaults (omitted asset and 0 amounts) if the transaction failed."
- `internal/transform/operation.go:69-81` — `OperationResultCode` and `OperationTraceCode` are populated from transaction results, enabling downstream consumers to distinguish failed operations.

### Why It Failed

**Duplicate of prior investigation.** The review of `024-failed-liquidity-pool-deposit-exports-zero-amounts.md` already examined the withdraw code path at line 1037 and concluded the entire pattern is working-as-designed. The same reasoning applies here: (1) explicit code comment documents intent, (2) consistent pattern across 6+ operation types, (3) zero amounts correctly reflect that no withdrawal occurred, (4) result codes enable downstream filtering. This withdraw hypothesis is the mirror image of the deposit hypothesis with no novel mechanism.

### Lesson Learned

When a prior investigation already explicitly traced the sibling code path and concluded working-as-designed, a hypothesis targeting that sibling path with the identical mechanism is a duplicate.
