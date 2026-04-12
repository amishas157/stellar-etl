# H024: `export_trades` zero-based `Parse(...)` path corrupts exported `history_operation_id`

**Date**: 2026-04-12
**Subsystem**: toid
**Severity**: Low
**Impact**: operational diagnostics
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Trade rows should export `history_operation_id` using the canonical one-based operation TOID, aligned with `history_operations.id`. Any command-side error context should describe the same operation ordinal so operators can correlate failures with the row shape that would otherwise be emitted.

## Mechanism

`input.GetTrades()` stores `OperationHistoryID` with the raw zero-based operation slot, while `TransformTrade()` normalizes that value with `outputOperationID := operationID + 1`. Because `export_trades` parses the pre-normalized `OperationHistoryID` only in its error branch, it initially looked like the command might be exporting or reporting an off-by-one operation TOID.

## Trigger

Force `TransformTrade()` to fail for a non-first trade-producing operation and compare the logged `(ledger, transaction, operation)` tuple against the `history_operation_id` that the successful path would emit.

## Target Code

- `internal/input/trades.go:64-70` — stores zero-based `OperationHistoryID`
- `internal/transform/trade.go:31-33,152` — normalizes to the exported one-based `HistoryOperationID`
- `cmd/export_trades.go:38-41` — parses the pre-normalized TOID only when logging transform failures

## Evidence

The same conceptual operation passes through two TOID representations: a zero-based staging ID in the input reader and a one-based canonical ID in `TransformTrade()`. `export_trades` calls `toid.Parse(tradeInput.OperationHistoryID)` on the staging value, not on the post-normalization output ID.

## Anti-Evidence

The parsed value is used only inside the `err != nil` logging branch. Successful JSON rows and any parquet staging rows use `TradeOutput.HistoryOperationID`, which comes from the normalized `outputOperationID` and is not overwritten by the parsed/logged value.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This discrepancy is confined to diagnostic text on failed transforms; it never mutates the exported trade row, so there is no silent data corruption in JSON or parquet output.

### Lesson Learned

Some trade code paths intentionally carry a zero-based staging TOID and a one-based exported TOID at the same time. Before escalating an off-by-one theory, confirm whether the suspicious value ever enters a schema field or remains in logging-only code.
