# H001: Token transfer operation IDs are off by one

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TokenTransferOutput.operation_id` should use the same TOID convention as the rest of the ETL: operation index 1 should produce the first operation TOID for the transaction, and later operations should preserve that 1-indexed sequence exactly.

## Mechanism

At first glance, `transformEvents()` looks suspicious because it feeds `event.Meta.OperationIndex` straight into `toid.New(...)` without the explicit `+1` adjustment used in `trade.go`. If `EventMeta.OperationIndex` were zero-indexed, token-transfer rows would be shifted by one operation and misjoin against the operations table.

## Trigger

Export token transfers for a transaction with at least one classic operation and compare the resulting `operation_id` to the operation TOID from `history_operations`.

## Target Code

- `internal/transform/token_transfer.go:transformEvents:82-91` — constructs `operation_id` directly from `event.Meta.OperationIndex`
- `internal/transform/trade.go:TransformTrade:31-34` — nearby transform code uses an explicit `+1` adjustment for operation IDs

## Evidence

The transform code paths are inconsistent on their face: `trade.go` compensates for ingest indexing, but `token_transfer.go` does not. That difference makes an off-by-one bug a natural first hypothesis.

## Anti-Evidence

The SDK `token_transfer.EventMeta` documentation says `OperationIndex` is already **1-indexed as per SEP-35**, and `internal/transform/token_transfer_test.go:56-149` expects `operation_id` `42949677057` from `OperationIndex=1`, matching the current implementation. That means the apparent inconsistency is explained by differing upstream conventions rather than a transform bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`token_transfer.EventMeta.OperationIndex` is already 1-indexed, so passing it directly to `toid.New(...)` produces the correct operation TOID.

### Lesson Learned

Nearby transform functions can use different indexing adjustments because they ingest different upstream abstractions. For token-transfer code, always verify the SDK metadata contract before treating a missing `+1` as an off-by-one bug.
