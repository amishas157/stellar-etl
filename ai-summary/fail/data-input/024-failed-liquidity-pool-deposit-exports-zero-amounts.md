# H001: Failed liquidity-pool deposits export zeroed reserve and share amounts

**Date**: 2026-04-12
**Subsystem**: data-input
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a `liquidity_pool_deposit` operation belongs to a failed transaction, `export_operations` should not emit a `history_operations` row claiming any realized reserve movement or pool-share issuance, because the ledger state did not change. At minimum, the exporter should not serialize `reserve_a_deposit_amount`, `reserve_b_deposit_amount`, and `shares_received` as if they were real executed amounts.

## Mechanism

`GetOperations()` expands every envelope operation without checking `tx.Result.Successful()`, and `export_operations` writes every transformed row it receives. In the `liquidity_pool_deposit` branch, `TransformOperation` only computes actual reserve deltas and shares when the transaction succeeded; on failure it keeps the zero-value placeholders and still exports the row, producing plausible-looking numeric fields of `0` for an operation that never executed.

## Trigger

1. Find a ledger range containing a failed `liquidity_pool_deposit` transaction.
2. Run `stellar-etl export_operations --start-ledger <L> --end-ledger <L> ...`.
3. Inspect the emitted row for that failed operation.
4. The exporter is expected to omit the row entirely, but the current path can emit a row whose `details.reserve_a_deposit_amount`, `details.reserve_b_deposit_amount`, and `details.shares_received` are all `0` even though no deposit occurred.

## Target Code

- `internal/input/operations.go:GetOperations:41-77` — expands every envelope operation and never filters failed transactions
- `cmd/export_operations.go:25-54` — writes every transformed operation row returned by `GetOperations()`
- `internal/transform/operation.go:69-98` — leaves `operation_result_code` empty when there is no per-operation result slice, but still returns a row
- `internal/transform/operation.go:957-1019` — `liquidity_pool_deposit` fills realized reserve/share fields only on success and otherwise exports zero defaults

## Evidence

The input layer has no `tx.Result.Successful()` guard, so failed transactions reach the operation transformer unchanged. Inside `TransformOperation`, the deposit branch initializes `depositedA`, `depositedB`, and `sharesReceived` to zero, and the code comment at line 975 explicitly says those defaults are used when the transaction failed. The row still includes request-side fields such as pool ID, reserve max amounts, and min/max price bounds, so the exported record looks like a real but zero-filled deposit rather than an omitted failed attempt.

## Anti-Evidence

If `export_operations` were intentionally defined as an export of attempted envelope operations rather than applied history operations, one could argue that failed operations deserve a row. But the schema labels this table as `history_operations`, and `LedgerOutput.OperationCount` is explicitly documented as counting only operations from successful transactions, so exporting zero-filled failed deposits still looks inconsistent with the surrounding data model.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full path from `GetOperations()` through `TransformOperation()` to `extractOperationDetails()` for the `LiquidityPoolDeposit` case. The code at line 974 explicitly checks `transaction.Result.Successful()` and uses zero defaults for failed transactions, with a deliberate comment: "we will use the defaults (omitted asset and 0 amounts) if the transaction failed." This same pattern is applied consistently across at least 5 other operation types (PathPaymentStrictReceive at line 641, PathPaymentStrictSend at line 682, LiquidityPoolWithdraw at line 1037, and their wrapper equivalents at lines 1390/1413/1620/1666). This is a deliberate, codebase-wide design choice, not an oversight.

### Code Paths Examined

- `internal/input/operations.go:GetOperations:30-81` — Confirmed: expands all envelope operations without success filtering. This is intentional — the input layer is a data reader, not a filter.
- `internal/transform/operation.go:30-100` — `TransformOperation` populates `OperationResultCode` and `OperationTraceCode` from the transaction result (lines 69-81), giving downstream consumers the ability to distinguish failed operations.
- `internal/transform/operation.go:957-1019` — The `LiquidityPoolDeposit` branch explicitly checks `transaction.Result.Successful()` at line 974, with a comment explaining the zero-default behavior for failed transactions.
- `internal/transform/operation.go:641-656` — `PathPaymentStrictReceive` uses the identical pattern: zero `source_amount` default, populated only on success.
- `internal/transform/operation.go:682-700` — `PathPaymentStrictSend` uses the identical pattern.
- `internal/transform/operation.go:1037-1080` — `LiquidityPoolWithdraw` uses the identical pattern.
- `internal/transform/schema.go:19-22` — `LedgerOutput.OperationCount` (successful-only) and `TxSetOperationCount` (all) are ledger-level summary fields, not constraints on what `export_operations` should output.

### Why It Failed

**Describes working-as-designed behavior.** The hypothesis treats the inclusion of failed operations in `export_operations` output as a bug, but the code explicitly and consistently handles failed transactions across all operation types that derive fields from results. Key reasons this is by-design:

1. **Explicit code comment** at line 975: "we will use the defaults (omitted asset and 0 amounts) if the transaction failed" — the developers intentionally coded this path.
2. **Consistent pattern** across 6+ operation types — this is not a one-off oversight but a systematic design choice mirroring Horizon's `history_operations` behavior (which also includes failed operations).
3. **Zero amounts are correct** — for a failed deposit, the deposited amounts genuinely ARE zero. No corruption occurs; the values accurately reflect the outcome.
4. **Result codes enable downstream filtering** — each row includes `OperationResultCode` and `OperationTraceCode`, allowing consumers to filter failed operations if desired.
5. **The `OperationCount` argument is misleading** — `LedgerOutput.OperationCount` and `TxSetOperationCount` are separate ledger-level summary fields serving different aggregation needs. Their existence does not constrain `export_operations` output scope.

### Lesson Learned

When a codebase explicitly handles a code path with comments explaining the design intent, and applies the same pattern consistently across many sibling branches, it is working-as-designed behavior. The presence of divergent counting semantics at a different aggregation level (ledger-level `OperationCount` vs operation-level export) does not imply a constraint on the export scope.
