# H014: `ReceiveParsedOrderbooks()` stops after the first batch

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the orderbook batch receiver is used to consume a bounded stream, it should keep parsing batches until the channel is closed so later ledgers in the requested range contribute their markets, accounts, offers, and events.

## Mechanism

`ReceiveParsedOrderbooks()` sets `batchRead = true` after parsing the first batch, then immediately breaks the outer loop on `if batchRead || orderbookChannel == nil`. That means any second or later batch on the same channel would be silently ignored, yielding partial orderbook state even though the producer kept sending more data.

## Trigger

1. Start `StreamOrderbooks()` over a range large enough to produce at least two batches.
2. Feed its channel into `ReceiveParsedOrderbooks()`.
3. Observe that only the first batch contributes parsed rows; later batches are left unread.

## Target Code

- `internal/input/orderbooks.go:240-265` — receiver exits once `batchRead` becomes true
- `internal/input/orderbooks.go:212-236` — bounded producer can emit multiple batches on the same channel

## Evidence

The receiver's only exit condition after a successful read is `batchRead || orderbookChannel == nil`, so the function cannot drain more than one batch from a still-open channel. The producer-side batching code is explicitly written to send multiple `OrderbookBatch` values for larger ranges.

## Anti-Evidence

Repository search shows `StreamOrderbooks()` and `ReceiveParsedOrderbooks()` are only referenced inside `internal/input/orderbooks.go`; no shipped CLI command or active export path currently uses this orderbook pipeline.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is a real control-flow bug in an unwired helper, but I could not find any current command, export path, or in-repo caller that reaches it. Without a live caller, it cannot corrupt any present stellar-etl output.

### Lesson Learned

Orderbook helpers in `internal/input/orderbooks.go` contain plausible data-loss bugs, but they remain out of scope until an exported command or other production path actually wires them in.
