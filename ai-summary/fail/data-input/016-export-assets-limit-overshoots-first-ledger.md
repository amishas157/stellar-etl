# H002: `export_assets --limit` can overshoot within the first ledger

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_assets --limit N` should export at most `N` assets, matching the user-facing limit flag text. If `N == 1`, the command should stop after the first qualifying asset instead of continuing through the rest of that ledger.

## Mechanism

`GetPaymentOperations()` appends every qualifying `Payment` / `ManageSellOffer` operation in the current ledger and only checks `limit` after the full ledger scan finishes. That makes the assets reader the outlier among sibling input readers: `GetOperations()`, `GetTrades()`, and `GetTransactions()` enforce their limits inside the inner read loops, but `GetPaymentOperations()` does not. A first ledger containing multiple distinct qualifying assets can therefore produce more rows than the requested limit.

## Trigger

Run `stellar-etl export_assets --limit 1 --start-ledger <L> --end-ledger <R>` where ledger `<L>` contains two qualifying operations that reference two different assets. The correct output is one asset row; this path will export both because the limit is only checked after the ledger finishes.

## Target Code

- `internal/utils/main.go:AddArchiveFlags:248-255` — user-facing flag contract says `--limit` is the maximum number of assets to export
- `internal/input/assets.go:GetPaymentOperations:31-58` — appends all qualifying operations in a ledger before checking `limit`
- `cmd/export_assets.go:30-69` — trusts the input slice to have already honored the user-visible limit
- `internal/input/operations.go:52-76` — sibling reader enforces its limit inside the inner loop, highlighting the missing check

## Evidence

The only limit check in `GetPaymentOperations()` is the post-ledger `if int64(len(assetSlice)) >= limit && limit >= 0 { break }`. There is no inner-loop guard after `append`, so the function can cross the requested bound before it ever tests the condition. The other input readers do have intra-ledger checks, which makes this omission look accidental rather than deliberate.

## Anti-Evidence

If the first scanned ledger contains at most `N` qualifying operations, or if multiple qualifying operations collapse to the same asset ID and are later deduplicated by `seenIDs`, the bug is harder to notice. The failure is still real on ledgers that surface more than `N` distinct assets early in the range.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-input/002-asset-readers-overshoot-limit-within-ledger.md.gh-published`
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same finding that was already confirmed and published as success entry 002 in the data-input subsystem. The success file documents both `GetPaymentOperations()` and `GetPaymentOperationsHistoryArchive()` overshooting `--limit` within a single ledger, with a PoC demonstrating 56 qualifying operations returned on mainnet ledger 30820015 with `limit=1`.

### Code Paths Examined

- `internal/input/assets.go:GetPaymentOperations:31-58` — already documented in success/data-input/002
- `internal/input/assets_history_archive.go:21-47` — also covered in the same success entry
- `internal/input/operations.go:52-77` — used as the sibling comparison in the same success entry

### Why It Failed

This is a direct duplicate of an already-confirmed and published finding. The success entry `002-asset-readers-overshoot-limit-within-ledger.md.gh-published` covers the identical root cause (outer-only limit check in asset readers), the identical trigger (multi-asset ledger with low limit), and the identical affected code paths.

### Lesson Learned

Check success directory for existing confirmed findings before generating hypotheses about limit enforcement in asset readers.
