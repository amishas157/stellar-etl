# H004: Failed path-payment-strict-send rows still export the requested source spend

**Date**: 2026-04-12
**Subsystem**: data-input
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a failed `path_payment_strict_send`, `export_operations` should not emit a `history_operations` row claiming that the sender spent the requested source amount, because the payment did not execute. If a failed attempt is retained, the row should not present `SendAmount` as a realized debit.

## Mechanism

`GetOperations()` expands failed transactions exactly like successful ones, and `export_operations` writes every transformed row. In the strict-send branch, `TransformOperation` always sets `details["source_amount"]` to the requested `SendAmount`, but only overwrites `details["amount"]` with the actual destination amount on success. A failed strict-send payment therefore exports a plausible-looking row saying the sender spent funds while the destination received `0`, even though no asset movement occurred.

## Trigger

1. Find a ledger range containing a failed `path_payment_strict_send` transaction.
2. Run `stellar-etl export_operations --start-ledger <L> --end-ledger <L> ...`.
3. Inspect the exported operation row.
4. The correct output is no successful payment row, but the current path can emit `details.source_amount=<requested send amount>` and `details.amount=0`.

## Target Code

- `internal/input/operations.go:GetOperations:41-77` — expands all envelope operations, including failed ones
- `cmd/export_operations.go:25-54` — exports every transformed operation row without a success filter
- `internal/transform/operation.go:69-98` — returns `OperationOutput` even when the transaction has no usable per-operation result
- `internal/transform/operation.go:1402-1423` — `path_payment_strict_send` unconditionally exports the requested source amount and only fills the realized destination amount on success

## Evidence

The strict-send branch sets `details["source_amount"] = amount.String(op.SendAmount)` before checking transaction success. The later success guard only populates the realized destination amount via `OperationResult()`, so failed rows preserve request metadata as if it were execution metadata. That creates silent monetary corruption for any downstream consumer summing sent amounts from `history_operations`.

## Anti-Evidence

The exporter may have been designed to preserve some request-side fields for debugging failed submissions. But the row is emitted through the `history_operations` pipeline and uses the same shape as successful operations, so there is no explicit marker that `source_amount` is merely an attempted spend rather than an applied on-chain amount.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantially equivalent to 024-failed-liquidity-pool-deposit-exports-zero-amounts.md (same design pattern)
**Failed At**: reviewer

### Trace Summary

Traced the `PathPaymentStrictSend` branch in both `extractOperationDetails` (line 660-699) and the wrapper `Details()` method (line 1402-1423). Confirmed that `source_amount` is set from `op.SendAmount` unconditionally, and `amount` is set to 0 by default, overwritten with `result.DestAmount()` only on success. This is the exact same pattern as `PathPaymentStrictReceive` (line 620-658) which sets `amount` from `op.DestAmount` unconditionally and `source_amount` to 0, overwritten on success. Both patterns are direct mirrors of the upstream Horizon operations processor at `stellar/go/services/horizon/internal/ingest/processors/operations_processor.go:380-397`.

### Code Paths Examined

- `internal/transform/operation.go:660-699` — `extractOperationDetails` for `PathPaymentStrictSend`: sets `source_amount = op.SendAmount` (envelope), `amount = 0` (default), overwrites `amount` with `result.DestAmount()` only on success. Intentional design.
- `internal/transform/operation.go:620-658` — `extractOperationDetails` for `PathPaymentStrictReceive`: symmetric pattern — sets `amount = op.DestAmount` (envelope), `source_amount = 0`, overwrites on success. Same design.
- `internal/transform/operation.go:1402-1423` — Wrapper `Details()` for strict-send: identical logic to `extractOperationDetails`.
- `internal/transform/operation.go:1379-1400` — Wrapper `Details()` for strict-receive: identical symmetric pattern.
- `stellar/go/services/horizon/internal/ingest/processors/operations_processor.go:380-397` — Upstream Horizon uses the **exact same code**: `details["source_amount"] = amount.String(op.SendAmount)` unconditionally, `details["amount"] = amount.String(result.DestAmount())` only on success.
- `internal/transform/operation.go:69-81` — `TransformOperation` populates `OperationResultCode` and `OperationTraceCode` from the transaction result, enabling downstream consumers to distinguish failed operations.

### Why It Failed

**Describes working-as-designed behavior** and is substantially equivalent to previously investigated hypothesis 024.

1. **Direct Horizon mirror**: The stellar-etl code is a copy of the upstream Horizon operations processor. The Horizon code at `operations_processor.go:380-397` uses the identical pattern — `source_amount` from envelope, `amount` from result on success only. This is Horizon's intentional data model for `history_operations`.

2. **Semantic correctness of envelope fields**: For `path_payment_strict_send`, `source_amount` IS the user-specified input parameter (how much to send), not a computed execution result. The user explicitly chose this amount. For `path_payment_strict_receive`, `amount` (destination) is the user-specified input. The "known" field always comes from the envelope; the "variable/unknown" field defaults to 0 on failure and is populated from the result on success.

3. **Symmetric design across both path payment types**: The pattern is consistent — strict-receive exports the requested destination amount with source=0 on failure; strict-send exports the requested source amount with destination=0 on failure. Treating one as a bug while accepting the other would be incoherent.

4. **Result codes enable downstream filtering**: Each row includes `OperationResultCode` and `OperationTraceCode`, providing clear failure signals. Any downstream consumer summing amounts should filter on these fields.

5. **Previously investigated**: Fail entry 024 (failed-liquidity-pool-deposit-exports-zero-amounts) established that including failed operations with envelope-derived amounts is by-design behavior. That review specifically noted PathPaymentStrictSend at line 682 as part of the same consistent pattern.

### Lesson Learned

When a hypothesis claims a field in a failed operation row is "wrong," verify whether that field represents an envelope input (user's request) versus an execution result. For path payments, the "fixed" side (source for strict-send, destination for strict-receive) is always the envelope value — it's correct by definition because the user specified it. The "variable" side correctly defaults to 0 on failure. This is Horizon's data model, not a bug.
