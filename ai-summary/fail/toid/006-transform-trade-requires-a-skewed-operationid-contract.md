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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the complete data flow from input layer through transform. `GetTrades()` (trades.go:69) and `GetAllHistory()` (all_history.go:77) both compute `OperationHistoryID` with the zero-based loop index: `toid.New(seq, tx.Index, index).ToInt64()`. The sole production caller `cmd/export_trades.go:38` passes this value to `TransformTrade()`, which adds 1 at line 33. Because the operation field occupies bits 0–11 of the TOID, `toid.New(L, T, 0).ToInt64() + 1` is arithmetically identical to `toid.New(L, T, 1).ToInt64()`. The exported `HistoryOperationID` (line 152) and synthetic offer IDs (line 119) both use this correctly incremented value, producing canonical 1-based operation TOIDs that match `TransformOperation` output.

### Code Paths Examined

- `internal/input/trades.go:56-71` — loop uses zero-based `index` from `range tx.Envelope.Operations()`, stores `toid.New(seq, tx.Index, index).ToInt64()` as `OperationHistoryID`
- `internal/input/all_history.go:61-79` — identical zero-based TOID construction for trades
- `cmd/export_trades.go:38` — sole production caller, passes `tradeInput.OperationHistoryID` directly
- `internal/transform/trade.go:33` — `outputOperationID := operationID + 1` converts zero-based to 1-based
- `internal/transform/trade.go:119` — `uint64(operationID)+1` for synthetic offer IDs, numerically equal to `outputOperationID`
- `internal/transform/trade.go:152` — `HistoryOperationID: outputOperationID` exports the canonical 1-based TOID
- `internal/transform/operation.go:32` — sibling comparison: `toid.New(ledgerSeq, tx.Index, operationIndex+1)` produces the same result via a different path
- `internal/transform/trade_test.go:172,740` — test passes `operationID=100`, expects `HistoryOperationID=101`, confirming the +1 contract
- `internal/toid/main.go:139-156` — TOID encoding: operation in lowest 12 bits confirms `int64 + 1` equals `opIndex + 1` in the constructor

### Why It Failed

The code produces **correct output for all existing callers**. The two-step approach (zero-based TOID at call site → +1 inside TransformTrade) yields the same canonical 1-based operation TOID as the single-step approach used by `TransformOperation` (`operationIndex+1` in the constructor). The arithmetic identity `toid.New(L,T,i).ToInt64() + 1 == toid.New(L,T,i+1).ToInt64()` holds for all valid operation indices (0 ≤ i < 4095). No production code path exists that passes an already-canonical (1-based) TOID to `TransformTrade()`. The hypothesis describes a maintainability trap for hypothetical future callers, which falls under the explicit out-of-scope criterion: "Theoretical issues without a concrete code path producing wrong output."

### Lesson Learned

When a function's internal `+1` adjustment looks like a bug because sibling functions handle the adjustment at the call site, verify the actual values flowing through all production callers. A different convention that produces correct output is a design choice, not a data corruption bug. The key test is whether any concrete code path produces wrong output today, not whether the API could be misused in the future.
