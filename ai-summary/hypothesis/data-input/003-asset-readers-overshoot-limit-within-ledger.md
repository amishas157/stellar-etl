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
