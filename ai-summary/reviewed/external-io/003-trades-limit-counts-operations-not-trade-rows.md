# H003: `export_trades --limit` caps trade-producing operations, not trade rows

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `export_trades`, the shared `--limit` flag is registered as "Maximum number of trades to export", so a run with `--limit N` should serialize at most `N` trade rows.

## Mechanism

`input.GetTrades()` increments `tradeSlice` once per trade-producing operation, then stops when `len(tradeSlice) >= limit`. But `transform.TransformTrade()` turns each selected operation into `[]TradeOutput` by iterating over every claimed offer in `claimedOffers`, and `cmd/export_trades.go` writes every returned row. A single manage-offer or path-payment operation that crosses multiple offers can therefore exceed the requested trade-row limit.

## Trigger

Run `export_trades --limit 1` against a ledger containing one successful offer-crossing operation with multiple fills. The CLI should stop after one trade row, but this path should export all trade rows produced by that one operation.

## Target Code

- `internal/utils/main.go:AddArchiveFlags:248-255` - CLI help text promises a maximum number of `trades`
- `internal/input/trades.go:GetTrades:25-85` - enforces the limit on `TradeTransformInput` items, one per qualifying operation
- `internal/transform/trade.go:20-42` - trade transform starts from a single operation
- `internal/transform/trade.go:131-160` - appends one `TradeOutput` per claimed offer
- `cmd/export_trades.go:28-71` - writes every trade row returned by `TransformTrade()`

## Evidence

`TransformTrade()` returns `[]TradeOutput`, not a single row, and its core loop appends once per `claimOffer`. Meanwhile `GetTrades()` only counts qualifying operations, so the CLI limit is applied before that one-to-many expansion happens.

## Anti-Evidence

Operations with exactly one claimed offer will not show the bug, and ledgers with no multi-fill trades will appear to honor the flag. The defect only surfaces on multi-claim trade operations.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

**Severity downgrade from High to Medium**: The individual trade rows are all correct — no data corruption occurs. The issue is that the limit control doesn't work as documented, which is an operational correctness problem rather than structural data corruption. Output consumers receive more rows than expected, but each row contains accurate data. This is a sibling of the already-reviewed effects limit bug (002), applying the same pattern to a different export command.

### Trace Summary

`GetTrades()` (input/trades.go:50-77) iterates transactions and operations, appending one `TradeTransformInput` per qualifying operation (manage buy/sell offer, create passive sell offer, path payment strict send/receive) to `tradeSlice`. The limit check `int64(len(tradeSlice)) >= limit` (line 73) counts these operations, not trade rows. `TransformTrade()` (trade.go:41-160) then expands each operation into `[]TradeOutput` by iterating over `claimedOffers` — one row per claimed offer in the operation result. `export_trades.go` (lines 46-58) writes every element of that slice to output with no secondary limit check.

### Code Paths Examined

- `internal/utils/main.go:AddArchiveFlags:250-254` — flag help text says "Maximum number of trades to export"
- `internal/input/trades.go:GetTrades:34-83` — limit enforced on `len(tradeSlice)`, which counts operations not trade rows
- `internal/input/trades.go:50` — outer loop condition `int64(len(tradeSlice)) < limit || limit < 0`
- `internal/input/trades.go:64-71` — appends one `TradeTransformInput` per qualifying operation
- `internal/input/trades.go:73` — inner limit check breaks on operation count, not row count
- `internal/transform/trade.go:34` — `extractClaimedOffers` returns slice of `xdr.ClaimAtom` (one-to-many)
- `internal/transform/trade.go:41-160` — loop over `claimedOffers`, appends one `TradeOutput` per offer
- `cmd/export_trades.go:37-59` — iterates all trade inputs, writes all resulting TradeOutput rows without limit check
- `internal/input/operations.go:52-70` — for comparison, `GetOperations` counts operations which map 1:1 to rows, so its limit works correctly

### Findings

The `--limit` flag for `export_trades` counts trade-producing operations, not emitted trade rows. A single manage-offer or path-payment operation can cross multiple offers in the order book, producing multiple `ClaimAtom` entries. `TransformTrade` expands each such operation into one `TradeOutput` per claimed offer. Since the limit is applied before this expansion and no secondary cap exists in the export command, `--limit N` can produce significantly more than N rows.

For contrast, `GetOperations()` counts operations against its limit and each operation maps to exactly one output row, so the limit works correctly there. The trades path is the outlier because of the one-to-many operation→trade expansion.

Note: The already-reviewed effects hypothesis (002) incorrectly stated that `GetTrades()` counts "its actual output entity against the limit." This review confirms that is not the case — `GetTrades()` counts operations, not trade rows.

### PoC Guidance

- **Test file**: `internal/input/trades_test.go` (or create a new integration-style test)
- **Setup**: Construct a mock `LedgerCloseMeta` containing one transaction with one manage-sell-offer operation that has 3+ `ClaimAtom` entries in its `OffersClaimed` result. This requires building an `xdr.TransactionResultPair` with a `ManageSellOfferSuccess` containing multiple claimed offers.
- **Steps**: Call `GetTrades(start, end, 1, env, false)` with `limit=1`. Then for each returned `TradeTransformInput`, call `TransformTrade()` and count the total number of `TradeOutput` rows.
- **Assertion**: Assert that `len(GetTrades result) == 1` (limit honored at operation level) BUT `total TradeOutput rows > 1` (limit NOT honored at trade row level), demonstrating the mismatch between the documented limit semantics and actual behavior.
