# H049: Contract-event operation IDs drift when earlier operations emit no events

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For mixed-operation transactions, `TransformContractEvent()` should assign each operation-scoped contract event the TOID for the actual emitting operation, even when earlier operations in the transaction emitted no contract events. Later event rows should not be shifted onto smaller operation indices simply because some previous operations have empty event lists.

## Mechanism

At first glance, `TransformContractEvent()` computes `operation_id` from the loop variable `i` over `transactionEvents.OperationEvents`, which looked risky if `OperationEvents` were a sparse list containing only eventful operations. That would have made `i+1` drift from the real operation index and silently assign wrong TOIDs to later contract-event rows.

## Trigger

Export contract events for a transaction where operation 1 emits no contract events and operation 2 does emit a contract event, then compare the emitted `operation_id` with the real operation index.

## Target Code

- `internal/transform/contract_events.go:41-55` — computes `operation_id` from the `OperationEvents` loop index
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/ingest/ledger_transaction.go:300-307` — `GetTransactionEvents()` allocates `OperationEvents` with `len(txMeta.Operations)` and preserves exact operation positions

## Evidence

The transform code itself does not independently verify that `i` is the real operation index; it trusts the shape of `transactionEvents.OperationEvents`. That made sparse packing a plausible source of silently wrong `operation_id` values for operation-scoped contract events.

## Anti-Evidence

The SDK implementation rejects the sparse-list premise: for V4 metadata it allocates one `OperationEvents` entry per operation and copies `op.Events` into the matching index, preserving empty slots for operations with no events. That keeps `i+1` aligned with the real operation number, so the suspected drift does not occur.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`GetTransactionEvents()` preserves one slot per transaction operation, so `TransformContractEvent()`'s `i+1` mapping is stable even when some operations emit zero contract events.

### Lesson Learned

Before flagging loop-index drift in transform code, inspect how the upstream ingest helper materializes the slice. A suspicious `for i := range events` pattern is only a bug if the upstream producer compacts away empty operation slots.
