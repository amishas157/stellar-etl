# H001: `export_assets --limit 0` still emits first-ledger asset rows

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Structural data corruption: a zero-row asset export can still produce non-empty output
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`--limit 0` should produce an empty `history_assets` export. The shared archive flag text defines `limit` as the maximum number of exported assets, so a caller explicitly asking for zero rows should get no JSON rows, no distinct asset IDs, and no uploaded artifact claiming assets were present in-range.

## Mechanism

Both asset readers append qualifying `payment` / `manage_sell_offer` operations for the current ledger before checking `len(assetSlice) >= limit`. When `limit == 0`, that post-ledger guard still fires, but only after the first in-range ledger has already contributed all of its qualifying operations. `cmd/export_assets` then deduplicates and writes those rows, turning an explicitly empty request into a plausible non-empty asset export.

## Trigger

1. Choose a ledger whose first in-range ledger contains at least one qualifying asset-discovery operation, such as ledger `30820015`.
2. Run `stellar-etl export_assets --start-ledger 30820015 --end-ledger 30820015 --limit 0 -o /tmp/assets.json`.
3. Inspect the JSON output.
4. The command should emit zero rows, but the current path will still write distinct asset rows from that first ledger.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:21-60` — datastore-backed asset reader appends qualifying operations before the only non-negative limit check.
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:13-50` — history-archive asset reader repeats the same outer-only limit enforcement.
- `cmd/export_assets.go:Run:30-80` — passes the caller-visible `limit` directly into the asset readers and writes every deduplicated row they return.
- `internal/utils/main.go:AddArchiveFlags:250-255` — documents `--limit` as the maximum number of exported assets.

## Evidence

The only limit check in both asset readers sits after the full per-ledger scan, unlike sibling readers that guard entry with `len(slice) < limit || limit < 0` and break immediately once the requested budget is exhausted. Because `export_assets` writes whatever distinct assets survive deduplication, any qualifying operation in the first ledger turns a `--limit 0` request into a non-empty export.

## Anti-Evidence

Published asset-reader findings already show the same post-ledger limit pattern for positive limits, so a reviewer may treat zero-limit as a boundary-value variant of an existing family rather than a distinct bug. But the observable behavior is materially different: this path converts an explicitly empty request into a non-empty asset file, matching the already-confirmed zero-limit bug class on ledger/token-transfer exports rather than merely overshooting by a few rows.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced both asset readers (`GetPaymentOperations` in `assets.go` and `GetPaymentOperationsHistoryArchive` in `assets_history_archive.go`). Both use unconditional `range` loops over transaction sets, appending all qualifying `Payment`/`ManageSellOffer` operations before a post-ledger limit check. With `limit=0`, the check `int64(len(assetSlice)) >= 0 && 0 >= 0` is trivially true (always), so the loop breaks after the first ledger — but only after all qualifying operations from that ledger have already been appended. Sibling readers (`transactions.go:51`, `operations.go:52`, `trades.go:50`) guard inner loop entry with `for int64(len(slice)) < limit || limit < 0`, which correctly prevents any appending when `limit=0`.

### Code Paths Examined

- `internal/input/assets.go:GetPaymentOperations:31-57` — outer `for seq` loop processes first ledger, inner `range transactionSet` / `range transaction.Operations()` appends unconditionally, post-ledger check at line 55 always true when limit=0 but items already appended
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:21-48` — identical pattern, same post-ledger-only enforcement at line 45
- `internal/input/transactions.go:51` — inner loop `for int64(len(txSlice)) < limit || limit < 0` correctly prevents entry when limit=0 (0 < 0 is false, 0 < 0 is false)
- `internal/input/operations.go:52` — same correct inner guard pattern
- `internal/input/trades.go:50` — same correct inner guard pattern
- `internal/input/ledgers.go:84-87` — same post-ledger-only pattern as assets (also affected for limit=0, but that's a separate finding)
- `cmd/export_assets.go:30-70` — passes limit to reader, then writes all returned items without secondary cap
- `internal/utils/main.go:250-254` — `AddArchiveFlags` documents limit as "Maximum number of assets to export", default -1

### Findings

The bug is confirmed. When `limit=0`:
1. `GetPaymentOperations()` enters the outer `for seq` loop for the first ledger
2. Inner `range` loops over transactions and operations unconditionally append all qualifying ops
3. Post-ledger check `int64(len(assetSlice)) >= 0 && 0 >= 0` is always true — breaks the loop
4. Returns a non-empty slice containing all qualifying ops from the first ledger
5. `export_assets` deduplicates and writes all of them to the output file

Sibling readers avoid this by guarding inner loop entry: `for int64(len(slice)) < limit || limit < 0` — when limit=0, `len < 0` is false and `0 < 0` is false, so no items are ever appended.

Severity downgraded from High to Medium: while the behavior clearly violates the documented contract, `--limit 0` is an uncommon boundary-value input. The bug produces extra rows rather than corrupted values, placing it in the "operational correctness" category.

### PoC Guidance

- **Test file**: `internal/input/assets_test.go` (if it exists) or create a unit test for `GetPaymentOperations`
- **Setup**: Create a mock ledger backend that returns one ledger with at least one qualifying Payment operation
- **Steps**: Call `GetPaymentOperations(start, start, 0, env, false)` with a limit of 0
- **Assertion**: Assert that the returned slice has length 0. Currently it will return a non-empty slice, proving the bug. Also test `GetPaymentOperationsHistoryArchive` with the same limit=0 argument.
