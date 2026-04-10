# H003: `export_trades --limit` counts candidate operations instead of emitted trade rows

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Structural data corruption: empty or truncated trade exports
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

The `--limit` flag for `export_trades` is documented as the maximum number of trades to export, so a positive limit should stop only after that many trade rows have actually been emitted. If the first successful offer-management operation in range produces no fills, `--limit 1` should continue scanning until it finds one actual trade row or reaches EOF.

## Mechanism

`input.GetTrades()` increments its `tradeSlice` for every successful operation whose type *can* result in trades, before `TransformTrade()` checks whether that operation actually claimed any offers. Successful manage-offer and passive-offer results can legally have `OffersClaimed == []`, and `TransformTrade()` then returns an empty slice with no error. Because the bounded reader has already exhausted the user-specified limit on that zero-trade operation, `export_trades` stops early and emits fewer rows than requested, including the fully empty case where the first limited operation posted an offer without matching anything.

## Trigger

Run `export_trades --limit 1` over a ledger range where the first successful `manage_buy_offer`, `manage_sell_offer`, or `create_passive_sell_offer` operation creates or updates an offer without matching any existing liquidity, and a later operation in the same range does execute a trade. The export will stop after the first operation and write zero trade rows instead of the later matching trade.

## Target Code

- `internal/utils/main.go:250-254` — `--limit` is described as "Maximum number of trades to export"
- `internal/input/trades.go:50-75` — the bounded loop counts trade-capable operations, not emitted trades
- `internal/transform/trade.go:34-41` — `claimedOffers` is derived only after the limit has already been consumed
- `internal/transform/trade.go:69-72, 187-220` — successful operations can yield zero `claimedOffers`, producing no output rows and no error
- `cmd/export_trades.go:37-58` — only transformed rows are written, so zero-row operations silently consume the limit

## Evidence

The reader appends one `TradeTransformInput` per successful operation matching `operationResultsInTrade()`, then checks `len(tradeSlice) >= limit` immediately in `internal/input/trades.go`. Later, `TransformTrade()` iterates `claimedOffers` to build rows; if `OffersClaimed` is empty, the function simply returns `[]TradeOutput{}` with no error. That creates a concrete mismatch between the user-facing "number of trades" contract and the actual stop condition.

## Anti-Evidence

Unlimited exports (`--limit < 0`) are unaffected because scanning continues through the full ledger range. The corruption only appears on bounded exports where zero-trade offer/path operations appear before later operations that would have produced actual trade rows.
