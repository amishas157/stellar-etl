# H004: TransformTrade silently shifts canonical operation TOIDs by one

**Date**: 2026-04-10
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a caller passes the canonical TOID for a successful trade-producing operation into `TransformTrade()`, the emitted `history_operation_id` should equal that exact TOID, and any synthetic `buying_offer_id` derived from it should encode the same operation. Trade rows should use the same operation identity convention as `history_operations` and operation-scoped contract events.

## Mechanism

The input layer currently computes `TradeTransformInput.OperationHistoryID` with the raw zero-based loop index, then `TransformTrade()` unconditionally does `operationID + 1` before populating `HistoryOperationID` and synthetic offer IDs. That hidden two-step contract means the function only behaves correctly for the specific callers that already pass a pre-skewed value; any caller that supplies the canonical TOID from `toid.New(ledger, tx.Index, opIndex+1)` will silently export IDs one operation too high.

## Trigger

Call `TransformTrade()` for a successful trade-producing operation with `operationID = toid.New(int32(seq), int32(tx.Index), int32(opIndex)+1).ToInt64()`, matching the canonical `history_operations.id` convention.

## Target Code

- `internal/input/trades.go:64-70` — `GetTrades()` stores `OperationHistoryID` using the zero-based operation index.
- `internal/input/all_history.go:72-78` — `GetAllHistory()` stores the same zero-based `OperationHistoryID`.
- `internal/transform/trade.go:31-33` — `TransformTrade()` immediately increments the caller-provided `operationID`.
- `internal/transform/trade.go:119-120` — synthetic offer IDs are also derived from the incremented value.
- `internal/transform/trade.go:152` — exported `HistoryOperationID` uses the incremented TOID.

## Evidence

Sibling exporters compute canonical operation IDs directly with `operationIndex+1` at the call site (`TransformOperation`, operation-scoped contract events), but `TransformTrade()` instead expects a pre-adjusted input and mutates it internally. The function signature and field name `OperationHistoryID` do not document that skew, so the wrong output remains perfectly plausible if a future caller passes the canonical TOID.

## Anti-Evidence

The current `cmd/export_trades` path is masked because it consumes `GetTrades()`, which already passes the zero-based value that `TransformTrade()` expects. A reviewer should decide whether the hidden API contract alone is enough to classify this as a live bug or only a maintainability trap.
