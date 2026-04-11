# H001: Token transfer operation IDs are off by one because transform code omits +1

**Date**: 2026-04-10
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TokenTransferOutput.operation_id` should use the same TOID convention as the rest of the ETL: the first operation-scoped token transfer in a transaction should get the first operation TOID for that transaction, and later operations should preserve that exact ordering.

## Mechanism

`TransformTokenTransfer()` passes `event.Meta.OperationIndex` directly into `toid.New(...)`, while `TransformOperation()` and operation-scoped contract events both add `+1` before encoding operation TOIDs. That looked like a classic off-by-one mismatch that would make token-transfer rows collide with transaction IDs for first operations and disagree with `history_operations.id`.

## Trigger

Export token transfers for a ledger containing operation-scoped events and compare `token_transfers.operation_id` with the corresponding `history_operations.id`.

## Target Code

- `internal/transform/token_transfer.go:82-90` — token-transfer transform passes `OperationIndex` directly into `toid.New(...)`.
- `internal/transform/operation.go:31-32` — operation export adds `+1` before encoding operation TOIDs.
- `internal/transform/contract_events.go:42-52` — operation-scoped contract events also add `+1`.

## Evidence

The local transform code is visibly inconsistent: token transfers omit the `+1` adjustment used by other operation-scoped exporters. That makes the path look like a silent TOID mismatch on first inspection.

## Anti-Evidence

The upstream SDK intentionally makes `token_transfer.EventMeta.OperationIndex` **1-indexed** in `processors/token_transfer/token_transfer_event.go:89-105`, and the local regression test `internal/transform/token_transfer_test.go:52-151` expects `OperationIndex=1` to encode directly to TOID `42949677057`. That means the apparent discrepancy is explained by different upstream index conventions, not a local transform bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`token_transfer.EventMeta.OperationIndex` arrives already SEP-35/TOID-aligned, so `TransformTokenTransfer()` is correct to pass it straight to `toid.New(...)`.

### Lesson Learned

When TOID callers disagree about whether `+1` is needed, verify the upstream index contract before treating the difference as a bug. In this codebase, ingest operation loops are zero-based, but token-transfer event metadata has already been normalized to 1-based operation numbering.
