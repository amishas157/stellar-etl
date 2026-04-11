# H001: `export_assets --limit` can return fewer distinct asset rows than requested

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_assets --limit N` should stop after exporting at most `N`
distinct asset rows, not after reading `N` raw operations that may later
collapse to fewer unique `asset_id` values. When the requested range contains at
least `N` distinct assets, the export should not silently return a smaller
result set just because the early operations reference the same asset multiple
times.

## Mechanism

Both asset readers stop when `len(assetSlice) >= limit`, but `assetSlice` counts
raw `Payment` / `ManageSellOffer` operations, not exported asset rows. The
command then applies a second-stage `seenIDs` deduplication in
`cmd/export_assets.go`, so repeated early references to asset A can exhaust the
reader limit before later distinct asset B is ever read, causing the final JSON
and Parquet outputs to contain fewer rows than the `--limit` contract promises.

## Trigger

1. Pick a ledger range whose first qualifying operations are:
   1. `Payment` or `ManageSellOffer` referencing asset A
   2. another qualifying operation referencing the same asset A
   3. a later qualifying operation referencing a distinct asset B
2. Run `stellar-etl export_assets --start-ledger <L> --end-ledger <R> --limit 2`.
3. The readers stop after the second raw operation, but `seenIDs` collapses the
   two A rows into one exported asset row, so asset B never appears even though
   the range contains two distinct assets.

## Target Code

- `internal/input/assets.go:31-56` — datastore reader enforces `limit` on raw qualifying operations
- `internal/input/assets_history_archive.go:21-46` — history-archive reader repeats the same raw-operation limit logic
- `cmd/export_assets.go:39-59` — export path deduplicates only after the reader limit has already truncated the slice
- `internal/utils/main.go:250-255` — shared flag text promises `limit` is the maximum number of exported `assets`

## Evidence

The limit check lives outside the inner operation loop in both asset readers and
uses `len(assetSlice)`, which grows once per qualifying operation. The command's
actual row contract is different: it writes only the first occurrence of each
`AssetID` after `TransformAsset()` and drops every duplicate through `seenIDs`.
That split makes row count depend on how many duplicate references happen to
appear before the reader hits the raw-operation limit.

## Anti-Evidence

If the first `N` qualifying operations already reference `N` distinct assets, or
if `--limit` is negative, the bug is masked because the post-read dedupe no
longer changes the cardinality. This does not affect ranges where the caller
expects duplicate asset references to collapse below the requested bound.
