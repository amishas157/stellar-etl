# H003: Asset Readers Overshoot `--limit` Within a Single Ledger

**Date**: 2026-04-10
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `stellar-etl export_assets --limit N` is used, the reader should stop after collecting at most `N` qualifying operations, matching the documented meaning of the limit flag and the exact-limit behavior used by the transaction, operation, and trade readers.

## Mechanism

Both asset readers append every qualifying payment or manage-sell-offer operation in the current ledger before checking whether `len(assetSlice) >= limit`. If the limit boundary is crossed inside the nested transaction/operation loops, the function returns more than `N` `AssetTransformInput` values, so `export_assets` can emit extra asset rows from the same ledger instead of honoring the requested cap.

## Trigger

Run `stellar-etl export_assets --limit 1 --start-ledger <L> --end-ledger <R>` where ledger `<L>` contains at least two qualifying operations for different assets. The export should return one asset input, but the current readers will keep scanning the rest of the ledger and can produce two or more rows before the outer per-ledger limit check fires.

## Target Code

- `internal/input/assets.go:31-57` — appends qualifying operations inside nested loops and checks `limit` only after finishing the ledger
- `internal/input/assets_history_archive.go:21-47` — repeats the same per-ledger limit check in the history-archive variant
- `cmd/export_assets.go:21-33` — exposes the `limit` flag as the caller-visible bound for asset exports
- `internal/input/operations.go:52-76` — sibling reader that stops inside the inner loop once the limit is reached
- `internal/input/trades.go:50-80` — sibling reader that also enforces the limit before finishing the ledger

## Evidence

Unlike `GetOperations()` and `GetTrades()`, neither asset reader has an inner-loop `break` when the slice reaches the requested limit. The only guard lives after the entire ledger has been traversed, so the overshoot can be as large as the number of additional qualifying operations in that ledger.

## Anti-Evidence

`export_assets` deduplicates repeated `AssetID` values before writing output, so overshoot is easiest to observe when the extra operations reference distinct assets. If the additional operations all point at an already-seen asset, the wrong reader count may not translate into more exported rows.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the limit enforcement in both asset readers (`GetPaymentOperations` and `GetPaymentOperationsHistoryArchive`) and compared against the sibling readers (`GetOperations` and `GetTrades`). The asset readers use a simple `for txIndex, transaction := range transactionSet` pattern with the limit check only at the outer ledger-iteration level (assets.go:55, assets_history_archive.go:45). In contrast, `GetOperations` (operations.go:52,67,75) and `GetTrades` (trades.go:50,73,80) each have three limit checks: (1) the outer read-loop condition, (2) an inner-loop break after each append, and (3) a post-ledger break. The asset readers are genuinely missing the inner-loop enforcement present in all sibling readers.

### Code Paths Examined

- `internal/input/assets.go:31-58` — Outer loop iterates ledgers; inner nested loops iterate `transactionSet` then `transaction.Operations()`. Qualifying ops are appended unconditionally. Limit check at line 55 fires only after both inner loops complete for the entire ledger.
- `internal/input/assets_history_archive.go:21-48` — Identical structure to `assets.go`. Same missing inner-loop limit enforcement at line 45.
- `internal/input/operations.go:52-78` — Outer loop condition `int64(len(opSlice)) < limit || limit < 0` (line 52) prevents reading the next transaction once limit is met. Inner loop break at line 67-69 stops mid-transaction. Post-ledger break at line 75. Three-tier enforcement.
- `internal/input/trades.go:50-83` — Same three-tier enforcement as operations: outer condition (line 50), inner break (line 73-75), post-ledger break (line 80).
- `cmd/export_assets.go:27-34` — Calls one of the two asset readers, passing `limit` directly. No post-reader truncation of the returned slice. The `seenIDs` deduplication (line 54) only removes duplicate AssetIDs, not excess results.

### Findings

The bug is confirmed: both asset readers (`GetPaymentOperations` and `GetPaymentOperationsHistoryArchive`) lack inner-loop limit checks that all sibling readers implement. The overshoot per-ledger is bounded by the number of qualifying operations (Payment + ManageSellOffer) in a single ledger, which could be up to 100,000 in the theoretical worst case (1000 txs × 100 ops).

Severity downgraded from High to Medium. The data in each exported row is correct — no fields are wrong, no values are corrupted. The issue is operational: the `--limit` flag does not bound output size as precisely as the sibling commands. This is an inconsistency in limit enforcement rather than structural data corruption. The `seenIDs` deduplication in `export_assets` further reduces visible impact, since overshoot only produces extra output rows when the extra operations reference distinct assets.

### PoC Guidance

- **Test file**: `internal/input/assets_test.go` (or create if absent; check existing test patterns in `internal/input/operations_test.go`)
- **Setup**: Construct a mock ledger backend containing a single ledger with ≥3 qualifying operations (Payment or ManageSellOffer) referencing distinct assets.
- **Steps**: Call `GetPaymentOperations(start, start, 1, env, false)` with `limit=1` on the crafted ledger.
- **Assertion**: Assert `len(result) == 1`. Currently this will fail — the result will contain all qualifying operations from the ledger (≥3), demonstrating the overshoot. Compare against `GetOperations(start, start, 1, env, false)` which correctly returns exactly 1 result.
