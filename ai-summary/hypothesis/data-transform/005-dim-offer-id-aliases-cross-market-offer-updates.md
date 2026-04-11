# H005: `DimOfferID` aliases cross-market updates of the same offer ID

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If an existing Stellar offer is updated onto a different asset pair, the normalized orderbook export should produce a new `offer_instance_id` because the corresponding `dim_offers` row has changed markets. A later `fact_offer_events` row for the updated offer should join to a `dim_offers` row whose `market_id` and asset semantics match the new market, not the old one.

## Mechanism

`extractDimOffer()` hashes only `offer.OfferID`, `offer.Amount`, and `offer.Price` into `DimOfferID`, even though the exported `DimOffer` row also contains `MarketID`, `MakerID`, `Action`, `BaseAmount`, `CounterAmount`, and `Price`. Stellar protocol semantics allow `ManageSellOffer` / `ManageBuyOffer` updates to reuse the same `offerID` while changing the selling and buying assets, so an offer can move from one market to another without changing its ID. When the numeric amount/price tuple is reused, `DimOfferID` stays constant, and `OrderbookParser` deduplicates the second `dim_offers` row away entirely because it keys `SeenOfferHashes` only by `DimOfferID`.

## Trigger

Use the same account and offer ID for two offer states with the same numeric tuple but different asset pairs, for example:

1. Create offer `offerID = X` on `XLM/USD` with `Amount = 500` and `Price = 2/1`
2. Update the same `offerID = X` onto `CUR1/CUR2` with `Amount = 500` and `Price = 2/1`
3. Export normalized orderbook history across both ledgers

The second fact row will still reference the first state's `offer_instance_id`, so downstream joins can resolve the later ledger event to the old market.

## Target Code

- `internal/transform/offer_normalized.go:139-167` — `DimOfferID` preimage omits `MarketID` and `Action`
- `internal/transform/schema.go:321-345` — `DimOffer` and `DimMarket` expose market-specific fields that should distinguish offer states
- `internal/input/orderbooks.go:97-107` — deduplicates emitted `dim_offers` rows solely by `DimOfferID`

## Evidence

The repository code makes the aliasing path concrete: `DimOfferID` is derived before `MarketID` and `Action` are even considered, and `OrderbookParser` never checks any other field before suppressing a later `dim_offers` row. Upstream protocol behavior makes the trigger reachable: Stellar docs describe `Offer ID` as `0` for new offers and "existing offer ID" to update or delete, and stellar-core's `OfferTests.cpp` contains a dedicated `"modify offer assets with liabilities"` section with subcases like `"selling native change both assets"` and `"buying native change both assets"` that call `market.updateOffer(acc1, offerID, {finalSelling, finalBuying, Price{2, 1}, 500})`.

## Anti-Evidence

One possible product interpretation is that `DimOfferID` is meant to represent offer lineage keyed mainly by Horizon `offerID`, not a fully normalized offer-state instance. But the fact table names the foreign key `offer_instance_id`, and the orderbook pipeline already keeps separate dimension rows for changed offer states when the hash changes, which strongly suggests state-instance semantics rather than lineage semantics.
