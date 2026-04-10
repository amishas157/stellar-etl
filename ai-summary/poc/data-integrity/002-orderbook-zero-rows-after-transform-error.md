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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/input/data_integrity_poc_test.go
**Test Name**: "TestOrderbookZeroRowsAfterTransformError"
**Test Language**: Go

### Demonstration

The test creates a batch of two offer changes — one valid created offer and one deleted offer — and feeds them to `parseOrderbook()`. The deleted offer causes `TransformOfferNormalized()` to return an error, leaving a zero-valued `NormalizedOfferOutput` in the preallocated slice. The iteration loop then marshals and appends this zero-valued struct, producing bogus dimension rows (`market_id:0`, `account_id:0`, `dim_offer_id:0`) and a phantom event (`ledger_id:0`). The test confirms all four zero-valued row types are present in the parser output.

### Test Body

```go
package input

import (
	"encoding/json"
	"strings"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestOrderbookZeroRowsAfterTransformError demonstrates that when
// TransformOfferNormalized fails for an offer in a batch (e.g., a deleted
// offer), parseOrderbook emits zero-valued dimension rows and events
// instead of skipping the failed entry.
func TestOrderbookZeroRowsAfterTransformError(t *testing.T) {
	logger := utils.NewEtlLogger()

	// Build a valid created offer
	testAccountID, _ := xdr.AddressToAccountId("GCEODJVUUVYVFD5KT4TOEDTMXQ76OPFOQC2EMYYMLPXQCUVPOB6XRWPQ")
	testAccount3ID, _ := xdr.AddressToAccountId("GBT4YAEGJQ5YSFUMNKX6BPBUOCPNAIOFAVZOF6MIME2CECBMEIUXFZZN")

	ethAsset := xdr.Asset{
		Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
		AlphaNum4: &xdr.AlphaNum4{
			AssetCode: xdr.AssetCode4{69, 84, 72, 0}, // "ETH"
			Issuer:    testAccount3ID,
		},
	}

	validOffer := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeOffer,
		Pre:        nil,
		Post: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: xdr.Uint32(100),
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: testAccountID,
					OfferId:  12345,
					Selling:  xdr.MustNewNativeAsset(),
					Buying:   ethAsset,
					Amount:   1000000000, // 100 XLM in stroops
					Price: xdr.Price{
						N: 1,
						D: 2,
					},
				},
			},
		},
	}

	// Build a deleted offer — triggers TransformOfferNormalized error
	deletedOffer := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryRemoved,
		Type:       xdr.LedgerEntryTypeOffer,
		Pre: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: xdr.Uint32(100),
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: testAccountID,
					OfferId:  99999,
					Selling:  xdr.MustNewNativeAsset(),
					Buying:   ethAsset,
					Amount:   500000000,
					Price: xdr.Price{
						N: 1,
						D: 3,
					},
				},
			},
		},
		Post: nil,
	}

	// Parse a batch containing one valid and one deleted offer
	parser := NewOrderbookParser(logger)
	batch := []ingest.Change{validOffer, deletedOffer}
	parser.parseOrderbook(batch, 42)

	// The bug: we expect only 1 valid offer's rows, but due to the zero-valued
	// slot from the failed transform, we get extra bogus rows.

	// Check for zero-valued market (market_id: 0)
	zeroMarketFound := false
	for _, m := range parser.Markets {
		if strings.Contains(string(m), `"market_id":0`) {
			zeroMarketFound = true
			break
		}
	}

	// Check for zero-valued account (account_id: 0)
	zeroAccountFound := false
	for _, a := range parser.Accounts {
		if strings.Contains(string(a), `"account_id":0`) {
			zeroAccountFound = true
			break
		}
	}

	// Check for zero-valued offer (dim_offer_id: 0)
	zeroOfferFound := false
	for _, o := range parser.Offers {
		if strings.Contains(string(o), `"dim_offer_id":0`) {
			zeroOfferFound = true
			break
		}
	}

	// Check for zero-valued event (ledger_id: 0)
	zeroEventFound := false
	for _, e := range parser.Events {
		var event map[string]interface{}
		if err := json.Unmarshal(e, &event); err == nil {
			if ledgerID, ok := event["ledger_id"]; ok {
				if ledgerID.(float64) == 0 {
					zeroEventFound = true
					break
				}
			}
		}
	}

	// Demonstrate the bug: zero-valued rows ARE present
	if !zeroMarketFound {
		t.Errorf("Expected zero-valued market row (market_id:0) from failed transform, but none found. Markets: %v", parser.Markets)
	}
	if !zeroAccountFound {
		t.Errorf("Expected zero-valued account row (account_id:0) from failed transform, but none found. Accounts: %v", parser.Accounts)
	}
	if !zeroOfferFound {
		t.Errorf("Expected zero-valued offer row (dim_offer_id:0) from failed transform, but none found. Offers: %v", parser.Offers)
	}
	if !zeroEventFound {
		t.Errorf("Expected zero-valued event row (ledger_id:0) from failed transform, but none found. Events: %v", parser.Events)
	}

	// Additional assertion: we should have gotten exactly 1 valid set of rows,
	// but due to the bug, we get 2 markets, 2 accounts, 2 offers, and 2 events
	if len(parser.Markets) != 2 {
		t.Errorf("Expected 2 markets (1 valid + 1 bogus zero), got %d", len(parser.Markets))
	}
	if len(parser.Accounts) != 2 {
		t.Errorf("Expected 2 accounts (1 valid + 1 bogus zero), got %d", len(parser.Accounts))
	}
	if len(parser.Offers) != 2 {
		t.Errorf("Expected 2 offers (1 valid + 1 bogus zero), got %d", len(parser.Offers))
	}
	if len(parser.Events) != 2 {
		t.Errorf("Expected 2 events (1 valid + 1 bogus zero), got %d", len(parser.Events))
	}

	t.Logf("Bug confirmed: parseOrderbook emits zero-valued rows for failed transforms")
	t.Logf("Markets: %d items", len(parser.Markets))
	for i, m := range parser.Markets {
		t.Logf("  Market[%d]: %s", i, string(m))
	}
	t.Logf("Accounts: %d items", len(parser.Accounts))
	for i, a := range parser.Accounts {
		t.Logf("  Account[%d]: %s", i, string(a))
	}
	t.Logf("Offers: %d items", len(parser.Offers))
	for i, o := range parser.Offers {
		t.Logf("  Offer[%d]: %s", i, string(o))
	}
	t.Logf("Events: %d items", len(parser.Events))
	for i, e := range parser.Events {
		t.Logf("  Event[%d]: %s", i, string(e))
	}
}
```

### Test Output

```
=== RUN   TestOrderbookZeroRowsAfterTransformError
time="2026-04-10T18:48:18.854-05:00" level=error msg="error json marshalling offer #1 in ledger sequence number #42: offer 99999 is deleted" pid=97999
    data_integrity_poc_test.go:157: Bug confirmed: parseOrderbook emits zero-valued rows for failed transforms
    data_integrity_poc_test.go:158: Markets: 2 items
    data_integrity_poc_test.go:160:   Market[0]: {"market_id":10357275879248593505,"base_code":"ETH","base_issuer":"GBT4YAEGJQ5YSFUMNKX6BPBUOCPNAIOFAVZOF6MIME2CECBMEIUXFZZN","counter_code":"native","counter_issuer":""}
    data_integrity_poc_test.go:160:   Market[1]: {"market_id":0,"base_code":"","base_issuer":"","counter_code":"","counter_issuer":""}
    data_integrity_poc_test.go:162: Accounts: 2 items
    data_integrity_poc_test.go:164:   Account[0]: {"account_id":4268167189990212240,"address":"GCEODJVUUVYVFD5KT4TOEDTMXQ76OPFOQC2EMYYMLPXQCUVPOB6XRWPQ"}
    data_integrity_poc_test.go:164:   Account[1]: {"account_id":0,"address":""}
    data_integrity_poc_test.go:166: Offers: 2 items
    data_integrity_poc_test.go:168:   Offer[0]: {"horizon_offer_id":12345,"dim_offer_id":7936886082871125416,"market_id":10357275879248593505,"maker_id":4268167189990212240,"action":"b","base_amount":100,"counter_amount":50,"price":0.5}
    data_integrity_poc_test.go:168:   Offer[1]: {"horizon_offer_id":0,"dim_offer_id":0,"market_id":0,"maker_id":0,"action":"","base_amount":0,"counter_amount":0,"price":0}
    data_integrity_poc_test.go:170: Events: 2 items
    data_integrity_poc_test.go:172:   Event[0]: {"ledger_id":42,"offer_instance_id":7936886082871125416}
    data_integrity_poc_test.go:172:   Event[1]: {"ledger_id":0,"offer_instance_id":0}
--- PASS: TestOrderbookZeroRowsAfterTransformError (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/input	0.761s
```
