# H005: Trade Operation ID Might Be Off By One

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_trades` should emit `history_operation_id` values that line up with the operation IDs used elsewhere in the ETL, so the same underlying operation can be joined consistently across trades and operations exports.

## Mechanism

At first glance, `GetTrades()` builds `OperationHistoryID` with `toid.New(..., index)` while `TransformTrade()` later computes `outputOperationID := operationID + 1`. That looks like a classic off-by-one path where the trade export might shift every operation ID by one compared with the rest of the system.

## Trigger

Export trades for any ledger containing matched offers, then compare the resulting `history_operation_id` against the corresponding operation ID from the operations export for the same operation.

## Target Code

- `internal/input/trades.go:GetTrades:64-70` — seeds `OperationHistoryID` from `toid.New(..., index)`
- `internal/transform/trade.go:31-34` — increments the provided ID by one before writing `HistoryOperationID`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/ingest/ledger_transaction.go:16-18` — documents that `LedgerTransaction.Index` is already 1-indexed
- `internal/transform/trade_test.go:171-174` — expects `TransformTrade(..., 100, ...)` to emit `HistoryOperationID == 101`

## Evidence

The trade path is the only place in this subsystem where the operation ID is split across input collection and a later `+1`, so it initially looks inconsistent with `TransformOperation`, which constructs the final TOID directly.

## Anti-Evidence

The upstream ingest package explicitly documents `LedgerTransaction.Index` as 1-indexed, and the trade transformer tests intentionally pass `100` in and assert that the exported `HistoryOperationID` is `101`. That means the `+1` is part of the established contract for trade IDs rather than an accidental shift.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The apparent mismatch is intentional: `GetTrades()` passes a base TOID whose operation component is still zero-based for that path, and `TransformTrade()` normalizes it to the one-based history operation ID that the tests expect. I did not find a concrete export where the resulting trade `history_operation_id` diverges from the operation ID for the same ledger/transaction/operation tuple.

### Lesson Learned

When TOID construction is split across reader and transformer layers, verify the upstream indexing contract before assuming an off-by-one bug. The ingest package's 1-indexed transaction order and the trade tests together show that this path is deliberate, not drift.
