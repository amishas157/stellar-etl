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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full path from `GetTrades()` in `internal/input/trades.go` through `TransformTrade()` in `internal/transform/trade.go` to the export loop in `cmd/export_trades.go`. Confirmed that `GetTrades()` counts `TradeTransformInput` elements (one per trade-capable operation) against the limit, not the actual `TradeOutput` rows that `TransformTrade()` later produces. The sibling functions `GetTransactions()` and `GetOperations()` have 1:1 input-to-output relationships so their limit semantics are accidentally correct, but `GetTrades()` has a 1:N (where N can be 0) relationship, making its limit semantics wrong.

### Code Paths Examined

- `internal/input/trades.go:50-83` — Loop condition `int64(len(tradeSlice)) < limit` counts `TradeTransformInput` elements (operations), not trade rows. Each matching operation appends exactly one element regardless of how many trades it produces.
- `internal/input/trades.go:64-71` — Filter appends any successful operation matching `operationResultsInTrade()`, which includes manage_buy_offer, manage_sell_offer, create_passive_sell_offer, path_payment_strict_send, and path_payment_strict_receive.
- `internal/transform/trade.go:21-161` — `TransformTrade()` extracts `claimedOffers` via `extractClaimedOffers()`. A successful manage offer can have `OffersClaimed == []` (offer posted but not matched), returning `[]TradeOutput{}` with nil error.
- `internal/transform/trade.go:164-262` — `extractClaimedOffers()` returns the `OffersClaimed` slice directly from the operation result. For manage offers, `success.OffersClaimed` can legitimately be empty when the offer is created/updated without matching.
- `cmd/export_trades.go:37-59` — Export loop iterates `trades` (the `TradeTransformInput` slice), calls `TransformTrade()`, and only writes the resulting `TradeOutput` rows. If `TransformTrade()` returns empty, nothing is written but the limit slot is consumed.
- `internal/utils/main.go:254` — `AddArchiveFlags` registers limit as "Maximum number of trades to export", clearly documenting output-row semantics.
- `internal/input/operations.go:52-77` and `internal/input/transactions.go:51-62` — Sibling functions use the same pattern but have 1:1 input-output ratios, so the semantic mismatch doesn't manifest there.

### Findings

The bug is confirmed. The limit is applied at the wrong granularity:

1. **Under-counting**: `--limit 5` with 5 operations that each have 0 claimed offers → 0 trade rows exported. User asked for 5 trades, got 0.
2. **Over-counting**: `--limit 5` with 5 operations that each have 3 fills → 15 trade rows exported. User asked for 5 trades, got 15.
3. **Worst case**: `--limit 1` where the first matching operation is an unmatched offer → completely empty export file despite trade data existing later in the range.

The default `--limit -1` (unlimited) is unaffected, so production pipelines that export full ranges work correctly. The bug only affects bounded exports.

### PoC Guidance

- **Test file**: `internal/input/trades_test.go` (or create a new integration-style test)
- **Setup**: Construct a mock ledger with two transactions: (1) a successful `manage_sell_offer` that creates an unmatched offer (OffersClaimed = []), and (2) a successful `manage_sell_offer` that matches and produces a trade (OffersClaimed has 1+ element). Use mock/test LedgerCloseMeta.
- **Steps**: Call `GetTrades(start, end, 1, env, useCaptiveCore)` with limit=1 over that range, then pass the result through `TransformTrade()`.
- **Assertion**: Assert that the returned `TradeTransformInput` slice has 1 element (as current code does), but `TransformTrade()` on that element returns an empty `[]TradeOutput{}`. This demonstrates the semantic mismatch: user asked for 1 trade but gets 0 rows. Alternatively, assert with limit=2 that both operations are consumed even though only the second produces trade output — showing the limit counts operations not trades.
