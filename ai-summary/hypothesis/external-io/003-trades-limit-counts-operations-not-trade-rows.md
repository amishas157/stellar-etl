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
