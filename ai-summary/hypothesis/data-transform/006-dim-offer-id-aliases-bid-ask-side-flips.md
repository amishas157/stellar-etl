# H006: `DimOfferID` aliases bid/ask side flips on the same canonical market

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If an existing offer keeps the same canonical market but swaps which asset it is selling versus buying, the normalized export should produce a new `offer_instance_id`. A bid (`action = "b"`) and an ask (`action = "s"`) on the same market are distinct orderbook states and should not collapse onto a single `dim_offers` row.

## Mechanism

`extractDimOffer()` derives `Action` from the selling asset after sorting the market assets, but `DimOfferID` is computed first from only `offer.OfferID`, `offer.Amount`, and `offer.Price`. That means the hash omits the very field that distinguishes bids from asks on a canonical market. When the same on-chain offer ID swaps `selling` and `buying` assets while keeping the same numeric amount/price tuple, the canonical `market_id` can stay the same but `Action`, `BaseAmount`, `CounterAmount`, and price orientation all change; `OrderbookParser` still suppresses the later row because the unchanged hash makes it look like the same offer instance.

## Trigger

Process two ledgers for the same account and offer ID:

1. Create `offerID = X` on `XLM/USD` with `Amount = 500` and `Price = 2/1`
2. Update the same `offerID = X` to `USD/XLM` with the same `Amount = 500` and `Price = 2/1`

The canonical market remains the same asset pair after sorting, but the normalized row should flip from bid to ask (or vice versa). The current export can instead keep a single `DimOfferID`, causing later `fact_offer_events` to join to the wrong side of the book.

## Target Code

- `internal/transform/offer_normalized.go:139-167` — computes `DimOfferID` without `Action`, then derives `Action` from the sorted assets
- `internal/transform/schema.go:321-329` — `DimOffer` explicitly exports `action`, `base_amount`, `counter_amount`, and `price` as part of the row identity
- `internal/input/orderbooks.go:97-107` — later offer rows are dropped whenever `DimOfferID` repeats

## Evidence

The code path is explicit: `Action` is computed from `sellingAsset == assets[0]`, but the hash preimage is already fixed before that branch. Stellar-core makes the trigger reachable with asset-swap update tests such as `"selling native swap assets"`, `"buying native swap assets"`, and `"non-native swap assets"`, all of which update an existing `offerID` with reversed selling/buying assets while reusing the same `Price{2, 1}` and amount tuple. On the ETL side, that means a later ask can reuse the same `offer_instance_id` as an earlier bid on the same market.

## Anti-Evidence

This hypothesis depends on the update preserving the same numeric amount/price tuple; if real-world clients usually change those values alongside the side flip, the alias becomes less common. There is also some overlap with the existing buy-side amount/price defects: if downstream consumers already special-case `action` when interpreting rows, they may notice the mismatch faster than they would a silent cross-market alias.
