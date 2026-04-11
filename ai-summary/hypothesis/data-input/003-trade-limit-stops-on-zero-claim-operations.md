# H003: Trade Limit Stops on Successful Operations That Produce Zero Trade Rows

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_trades --limit 1` should keep reading until it can emit one actual trade row. A successful offer-management or path-payment operation that matches no offers should not consume the caller's single requested trade, because the correct exported row count is based on realized trades, not merely on operation types that are *capable* of trading.

## Mechanism

`GetTrades()` increments its limit accounting as soon as it sees a successful operation type from `operationResultsInTrade()`, but it does not inspect whether that operation actually produced any claimed offers. `TransformTrade()` later iterates `claimedOffers` and legitimately returns an empty slice when the operation only posts to the book, so the input reader can exhaust `--limit` on a zero-row operation and prevent later real trades in the range from ever being read.

## Trigger

Run `stellar-etl export_trades --limit 1 --start-ledger <S> --end-ledger <E>` on a range where the first successful manage-buy/manage-sell/create-passive/path-payment operation has `OffersClaimed == []`, but a later operation in the same range executes a real trade. The correct output is one trade row from the later operation; the current code path can return an empty file instead.

## Target Code

- `internal/input/trades.go:GetTrades:25-85` — counts candidate operations against `limit` before knowing whether they generate any trade rows
- `internal/input/trades.go:operationResultsInTrade:88-104` — classifies by operation type only
- `internal/transform/trade.go:39-160` — returns zero `TradeOutput` rows when `claimedOffers` is empty
- `cmd/export_trades.go:28-59` — trusts `GetTrades()` to have already enforced the user-visible `--limit`

## Evidence

The inline comment in `GetTrades()` explicitly says "Not all of these operations will result in trades," which confirms the reader is intentionally appending inputs that may expand to zero rows later. Because the limit check lives in `GetTrades()` instead of after `TransformTrade()`, the first successful zero-claim operation can terminate the scan before any real trade row is exported.

## Anti-Evidence

If the first counted operation actually claims at least one offer, the command will still emit data, so the bug only appears on ranges where non-matching successful trade-capable operations come first. Users who leave `--limit` negative also avoid the truncation because the reader keeps scanning the full range.
