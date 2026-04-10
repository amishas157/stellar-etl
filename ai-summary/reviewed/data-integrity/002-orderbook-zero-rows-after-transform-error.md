# H002: Orderbook parser emits zero-valued rows after a failed offer transform

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `TransformOfferNormalized()` fails for an offer change, that offer should be skipped entirely: no `dim_markets`, `dim_accounts`, `dim_offers`, or `fact_offer_events` rows should be emitted for that failed input. Failed transformations should reduce coverage, not fabricate placeholder records.

## Mechanism

`parseOrderbook()` preallocates `allConverted` and launches `convertOffer()` goroutines to fill each index. When a transform fails, `convertOffer()` only logs the error and leaves that slot at its Go zero value; `parseOrderbook()` then iterates every slot unconditionally and marshals the zero-valued `Market`, `Account`, `Offer`, and `Event`. A single rejected offer can therefore create bogus rows with ID `0`, empty strings, and `ledger_id=0`, which look structurally valid enough to poison downstream orderbook tables.

## Trigger

Feed `parseOrderbook()` a batch containing any offer change that makes `TransformOfferNormalized()` return an error, such as a deleted offer (rejected at `TransformOfferNormalized()` before normalization). The batch output will contain one synthetic zero row for the failed slot.

## Target Code

- `internal/input/orderbooks.go:37-45` — `convertOffer()` logs transform errors but does not mark the slot invalid
- `internal/input/orderbooks.go:62-117` — `parseOrderbook()` marshals every `allConverted` entry, including untouched zero values
- `internal/transform/offer_normalized.go:16-26` — deleted offers are a concrete path that returns an error instead of a normalized record

## Evidence

`allConverted := make([]transform.NormalizedOfferOutput, len(orderbook))` initializes every slot to zero values. On error, `convertOffer()` skips `allConvertedOffers[index] = transformed`, so the later loop sees a default `NormalizedOfferOutput{}` and still appends its marshaled children unless a later marshal itself fails.

## Anti-Evidence

The seen-hash maps limit the damage after the first zero-valued market/account/offer is inserted, so repeated failures will tend to reuse the same bogus IDs rather than creating infinitely many variants. That still leaves at least one bad dimension row plus zero-valued events for each failed slot.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full path from `parseOrderbook()` through `convertOffer()` to `TransformOfferNormalized()`. Confirmed that `allConverted` is preallocated with zero-valued `NormalizedOfferOutput` structs (line 64). When `TransformOfferNormalized()` fails — concretely via a deleted offer hitting line 25 of `offer_normalized.go` — `convertOffer()` only logs the error (lines 40-42) and leaves the slot untouched. The subsequent iteration loop (lines 72-118) has no guard against zero-valued entries, so it marshals and appends bogus dimension rows with `ID=0` and empty strings, plus a `FactOfferEvent{LedgerSeq: 0, OfferInstanceID: 0}` for every failed slot.

### Code Paths Examined

- `internal/input/orderbooks.go:64` — `allConverted := make([]transform.NormalizedOfferOutput, len(orderbook))` preallocates zero-valued structs
- `internal/input/orderbooks.go:37-46` — `convertOffer()` on error: logs error, does NOT set `allConvertedOffers[index]`, slot remains zero
- `internal/input/orderbooks.go:72-118` — iteration loop processes ALL entries; no nil/zero check before marshalling
- `internal/input/orderbooks.go:73` — `SeenMarketHashes[0]` will not exist on first zero entry → bogus `DimMarket{ID:0}` appended
- `internal/input/orderbooks.go:85` — `SeenAccountHashes[0]` same pattern → bogus `DimAccount{ID:0, Address:""}` appended
- `internal/input/orderbooks.go:97` — `SeenOfferHashes[0]` same pattern → bogus `DimOffer{DimOfferID:0, HorizonID:0}` appended
- `internal/input/orderbooks.go:110-116` — `FactOfferEvent` has NO dedup check → every failed slot emits `{LedgerSeq:0, OfferInstanceID:0}`
- `internal/transform/offer_normalized.go:24-25` — `if transformed.Deleted` returns error, a concrete trigger path
- `internal/utils/main.go:858-859` — `ExtractEntryFromChange` for REMOVED changes returns `(Pre, type, true, nil)`, setting `Deleted=true`
- `internal/transform/schema.go:315-352` — Zero values for all dimension structs: `uint64(0)` for IDs, `""` for strings, `0.0` for floats

### Findings

The bug is confirmed. When `TransformOfferNormalized()` returns an error for any offer change:

1. **One bogus `DimMarket`** is created with `{ID: 0, BaseCode: "", BaseIssuer: "", CounterCode: "", CounterIssuer: ""}` (deduped after first insertion via `SeenMarketHashes[0]`)
2. **One bogus `DimAccount`** is created with `{ID: 0, Address: ""}` (deduped after first)
3. **One bogus `DimOffer`** is created with `{DimOfferID: 0, HorizonID: 0, MarketID: 0, MakerID: 0, Action: "", BaseAmount: 0, CounterAmount: 0, Price: 0}` (deduped after first)
4. **One `FactOfferEvent` per failed slot** with `{LedgerSeq: 0, OfferInstanceID: 0}` — NOT deduped, so N failures create N bogus events

Deleted offers are a concrete and realistic trigger: any offer removal in the orderbook change set passes through `ExtractEntryFromChange` → `Deleted=true` → `TransformOfferNormalized` returns error at line 25. This is not an edge case; offer deletions are routine on the Stellar network.

The downstream impact is injection of structurally valid but semantically meaningless rows into BigQuery `dim_markets`, `dim_accounts`, `dim_offers`, and `fact_offer_events` tables. These zero-ID rows could corrupt joins, skew analytics aggregations, and create phantom offer activity.

### PoC Guidance

- **Test file**: `internal/input/orderbooks_test.go` (create if not exists; or append to existing test file in `internal/input/`)
- **Setup**: Create an `OrderbookParser` via `NewOrderbookParser()`. Construct a batch of `ingest.Change` objects where at least one is a `LedgerEntryChangeTypeLedgerEntryRemoved` offer change (triggers `Deleted=true` path). Mix with one valid created/updated offer change.
- **Steps**: Call `parseOrderbook(batch, seq)` on the parser.
- **Assertion**: Assert that `len(parser.Events)` equals the number of successful transforms only (should be 1, not 2). Assert that no element in `parser.Markets` contains `"market_id":0`. Assert that no element in `parser.Accounts` contains `"account_id":0`. Assert that no element in `parser.Offers` contains `"dim_offer_id":0`.
