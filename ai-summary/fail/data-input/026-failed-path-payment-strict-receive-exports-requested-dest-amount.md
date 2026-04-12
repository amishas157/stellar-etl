# H003: Failed path-payment-strict-receive rows still export the requested destination amount

**Date**: 2026-04-12
**Subsystem**: data-input
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a failed `path_payment_strict_receive`, `export_operations` should not emit a `history_operations` row showing a delivered destination amount, because the payment never completed. If a failed attempt is exported at all, the row should not claim that the destination received `op.DestAmount`.

## Mechanism

The input reader exports all envelope operations, including ones from failed transactions. In the strict-receive branch, `TransformOperation` unconditionally sets `details["amount"]` to the requested `DestAmount`, but only fills `details["source_amount"]` from the operation result when the transaction succeeded. For failed transactions, the exporter therefore emits a row that says the recipient got the full requested amount while the source spent `0`, which is a materially wrong payment record.

## Trigger

1. Find a ledger range containing a failed `path_payment_strict_receive` transaction.
2. Run `stellar-etl export_operations --start-ledger <L> --end-ledger <L> ...`.
3. Inspect the emitted operation row.
4. The correct output is no successful payment row, but the current path can emit `details.amount=<requested dest amount>` and `details.source_amount=0`.

## Target Code

- `internal/input/operations.go:GetOperations:41-77` — surfaces failed-transaction operations to the exporter
- `cmd/export_operations.go:25-54` — serializes every transformed operation row
- `internal/transform/operation.go:69-98` — does not reject failed transactions before returning `OperationOutput`
- `internal/transform/operation.go:1380-1400` — `path_payment_strict_receive` always exports the requested destination amount, and only patches in the true source amount on success

## Evidence

The strict-receive branch sets `details["amount"] = amount.String(op.DestAmount)` before any success check. Only `details["source_amount"]` is guarded by `if operation.transaction.Result.Successful()`, so a failed payment produces a row whose financial fields no longer describe any real on-chain transfer. Because the row still includes normal-looking source, destination, asset, and path metadata, downstream analytics can silently treat it as an executed payment.

## Anti-Evidence

One could argue that `amount` is meant to preserve the requested destination amount, not the realized amount. But the command exports `history_operations`, not raw submission envelopes, and the same branch already tries to populate realized execution data from `OperationResult()` on success, which suggests the exported fields are intended to describe applied outcomes rather than user intent alone.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (though closely related to fail/024 and fail/025 which cover the same design pattern for other operation types)
**Failed At**: reviewer

### Trace Summary

Traced the `path_payment_strict_receive` details extraction in both `extractOperationDetails` (line 631) and the `Details()` method (line 1384). Both unconditionally set `details["amount"] = op.DestAmount` and default `details["source_amount"] = 0`, then only populate `source_amount` from results on success. Critically, verified that the upstream **Horizon `operations_processor.go`** (stellar/go) uses the **identical pattern** at lines 356-368: `details["amount"] = amount.String(op.DestAmount)` unconditionally, with `source_amount` populated from results only when `Successful()`. This confirms the ETL intentionally mirrors Horizon's `history_operations` semantics.

### Code Paths Examined

- `internal/transform/operation.go:620-658` — `extractOperationDetails` for PathPaymentStrictReceive: sets `amount = op.DestAmount` (line 631), `source_amount = 0` (line 632), updates `source_amount` from result only on `Successful()` (line 641-656)
- `internal/transform/operation.go:1379-1400` — `Details()` method for PathPaymentStrictReceive: identical pattern — `amount = op.DestAmount` (line 1384), `source_amount = 0` (line 1385), updates on success (lines 1390-1393)
- `internal/transform/operation.go:660-697` — PathPaymentStrictSend: mirror pattern — `amount = 0` default (line 672), `source_amount = op.SendAmount` unconditional (line 673), `amount` updated from result on success (line 696). Confirms consistent design: the "known" side (user's request) is always populated, the "computed" side defaults to 0 and is filled on success.
- `internal/transform/operation.go:1402-1416` — PathPaymentStrictSend `Details()`: same mirror pattern
- `stellar/go@.../processors/operations_processor.go:351-368` — **Horizon upstream**: EXACT same code — `details["amount"] = amount.String(op.DestAmount)` unconditionally (line 356), `source_amount` from result only on `Successful()` (lines 366-368)
- `internal/input/operations.go:30-81` — GetOperations: expands all envelope operations without success filtering (intentional — input layer is a reader, not a filter)
- `internal/transform/operation.go:69-81` — TransformOperation populates `OperationResultCode` and `OperationTraceCode` from result, enabling downstream consumers to distinguish failed from successful operations

### Why It Failed

**Describes working-as-designed behavior that exactly mirrors Horizon's upstream implementation.** The hypothesis misinterprets the `amount` field semantics for `path_payment_strict_receive`:

1. **`amount` = requested destination amount, not delivered amount.** In a strict-receive path payment, the user specifies the exact destination amount they want delivered. The `amount` field preserves this envelope-level request. On success, the requested amount IS the delivered amount (that's what "strict receive" means). The field is not claiming delivery occurred for failed transactions.

2. **Mirrors Horizon exactly.** The upstream `stellar/go` Horizon `operations_processor.go` (line 356) uses the identical code: `details["amount"] = amount.String(op.DestAmount)` unconditionally, with `source_amount` from result only on success. This is the canonical Stellar `history_operations` schema.

3. **Symmetric design with StrictSend.** PathPaymentStrictSend mirrors this: `source_amount = op.SendAmount` unconditionally, `amount = 0` default updated on success. The pattern is: populate the user-specified side from the envelope, populate the computed side from the result only on success. This is consistent and intentional.

4. **Result codes enable downstream filtering.** Each row includes `OperationResultCode` and `OperationTraceCode`, allowing consumers to distinguish failed operations. Downstream systems that care about success-only data can filter on these fields.

5. **Consistent with fail/024 and fail/025.** This is the same codebase-wide design pattern already analyzed for `liquidity_pool_deposit` (fail/024) and `liquidity_pool_withdraw` (fail/025) — failed operations are exported with envelope data and zero/default results.

### Lesson Learned

When a hypothesis claims the ETL exports wrong financial data for failed operations, verify the upstream Horizon `operations_processor.go` first. The stellar-etl intentionally mirrors Horizon's `history_operations` semantics, where the `amount` field for `path_payment_strict_receive` always contains the requested destination amount from the envelope — not a claim about actual delivery. The distinction between "requested amount" and "delivered amount" is moot for strict-receive operations because they are identical on success.
