# H011: Token-transfer fee events do not currently need an operation TOID

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A token-transfer row should only contain `operation_id` when the upstream event is operation-scoped. If the upstream processor models fee charges and refunds as transaction-scoped events, the ETL should preserve `operation_id = NULL` rather than invent a derived TOID.

## Mechanism

The tempting hypothesis was that fee events from single-operation Soroban transactions should inherit that lone operation's TOID, because `TransformTokenTransfer()` leaves `OperationID` null whenever `EventMeta.OperationIndex` is absent. That looked like a silent loss of a join key for fee/refund rows.

## Trigger

Trace the token-transfer fee-event generation path, then inspect how `TransformTokenTransfer()` handles events whose metadata omits `OperationIndex`.

## Target Code

- `internal/transform/token_transfer.go:82-91` — leaves `OperationID` null when `event.Meta.OperationIndex == nil`.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/processors/token_transfer/token_transfer_processor.go:85-142` — fee events are modeled as ledger/transaction-level events that are ordered separately from operation events.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/processors/token_transfer/token_transfer_processor.go:217-265` — transaction processing builds `FeeEvents` separately from `OperationEvents`.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/processors/token_transfer/token_transfer_processor_test.go:301` — fee events are created with `NewEventMetaFromTx(..., nil, ...)`, i.e. no operation index by design.

## Evidence

`TransformTokenTransfer()` will happily export a null `operation_id` for fee rows even when the surrounding transaction has just one operation, so the missing join key is visible in the final JSON. Without checking the upstream processor, that looks like a local omission.

## Anti-Evidence

The upstream token-transfer processor explicitly treats fee events as a separate transaction-level stream and constructs their metadata with `operationIndex == nil`. That means the ETL is preserving upstream scope correctly rather than dropping an operation TOID that should exist.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated in `toid`

### Why It Failed

Fee and fee-refund token-transfer events are intentionally transaction-scoped in the upstream processor, so a null `operation_id` is the correct output. Deriving a synthetic operation TOID locally would invent scope that the source event model does not claim.

### Lesson Learned

Do not "upgrade" a null TOID just because the surrounding transaction has one operation. First confirm whether the upstream event producer treats the row as operation-scoped or transaction-scoped; for token-transfer fee events, the nil operation index is explicit design.
