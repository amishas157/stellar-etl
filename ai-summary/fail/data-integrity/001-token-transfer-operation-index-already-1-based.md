# H001: Token Transfer OperationIndex Already Matches Exported TOID

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformTokenTransfer()` should generate the same `operation_id` that the rest of the ETL uses for the corresponding history operation. If the upstream token-transfer metadata were zero-based, the transformer would need to add `+1` before calling `toid.New(...)` so token-transfer rows align with `history_operations`.

## Mechanism

The suspected bug was that `transformEvents()` passes `eventMeta.OperationIndex` straight into `toid.New(...)`, while other code paths such as `transactionOperationWrapper.ID()` add `+1` to local zero-based indexes. That would have produced token-transfer `operation_id` values one step behind the rest of the export surface.

## Trigger

1. Inspect a token-transfer event with a non-nil `OperationIndex`.
2. Compare the token-transfer `operation_id` formula with the operation-id formula used elsewhere.
3. Check whether the upstream metadata is zero-based or already one-based.

## Target Code

- `internal/transform/token_transfer.go:86-90` — token-transfer path feeds `OperationIndex` directly into `toid.New(...)`
- `internal/transform/operation.go:1185-1191` — local operation wrapper adds `+1` to a zero-based loop index
- `internal/transform/token_transfer_test.go:56-59` — expected token-transfer output uses operation id `42949677057`
- `internal/transform/token_transfer_test.go:157-167` — test input already supplies `OperationIndex = 1`

## Evidence

At first glance, the lack of `+1` in `token_transfer.go` looks suspicious because the operation exporter computes IDs from zero-based loop indexes and explicitly increments them. The same subsystem therefore contains both patterns side by side.

## Anti-Evidence

The token-transfer unit test shows the upstream `EventMeta.OperationIndex` is already one-based: input `OperationIndex = 1` produces expected `OperationID = 42949677057`, which is the first operation TOID for ledger 10 / tx 1. The `+1` in `operation.go` compensates for a local Go slice index, not for an upstream metadata contract.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`token_transfer.EventMeta.OperationIndex` is already encoded in the exported one-based operation numbering, so using it directly is correct and keeps token-transfer rows aligned with the test oracle.

### Lesson Learned

Do not assume every `OperationIndex` in the repository shares the same origin. Local loop counters often need `+1`, but upstream metadata fields may already be normalized to ETL-facing TOIDs.
