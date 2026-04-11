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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTradesLimitCountsOperationsNotTradeRows"
**Test Language**: Go

### Demonstration

The test constructs a single ManageSellOffer operation with 3 ClaimAtom entries (3 offers crossed) and calls `TransformTrade`. It proves that one operation — which `GetTrades` would count as 1 item toward `--limit` — expands into 3 TradeOutput rows. Since `export_trades.go` writes all returned rows without a secondary limit check, `--limit 1` would emit 3 trade rows instead of the documented maximum of 1.

### Test Body

```go
package transform

import (
	"testing"
	"time"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestTradesLimitCountsOperationsNotTradeRows demonstrates that a single
// trade-producing operation can yield multiple TradeOutput rows. Since
// GetTrades enforces --limit by counting operations (TradeTransformInput),
// not output rows, --limit N can produce significantly more than N trade rows
// when an operation crosses multiple offers.
func TestTradesLimitCountsOperationsNotTradeRows(t *testing.T) {
	// 1. Construct a single ManageSellOffer operation that claims 3 offers.
	//    GetTrades would count this as 1 TradeTransformInput toward the limit.
	offer1 := xdr.ClaimAtom{
		Type: xdr.ClaimAtomTypeClaimAtomTypeOrderBook,
		OrderBook: &xdr.ClaimOfferAtom{
			SellerId:     testAccount1ID,
			OfferId:      100,
			AssetSold:    ethAsset,
			AssetBought:  usdtAsset,
			AmountSold:   5000000,
			AmountBought: 1000,
		},
	}
	offer2 := xdr.ClaimAtom{
		Type: xdr.ClaimAtomTypeClaimAtomTypeOrderBook,
		OrderBook: &xdr.ClaimOfferAtom{
			SellerId:     testAccount2ID,
			OfferId:      200,
			AssetSold:    ethAsset,
			AssetBought:  usdtAsset,
			AmountSold:   3000000,
			AmountBought: 600,
		},
	}
	offer3 := xdr.ClaimAtom{
		Type: xdr.ClaimAtomTypeClaimAtomTypeOrderBook,
		OrderBook: &xdr.ClaimOfferAtom{
			SellerId:     testAccount3ID,
			OfferId:      300,
			AssetSold:    ethAsset,
			AssetBought:  usdtAsset,
			AmountSold:   2000000,
			AmountBought: 400,
		},
	}

	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: testAccount4,
			Memo:          xdr.Memo{},
			Operations: []xdr.Operation{
				{
					SourceAccount: nil,
					Body: xdr.OperationBody{
						Type:              xdr.OperationTypeManageSellOffer,
						ManageSellOfferOp: &xdr.ManageSellOfferOp{},
					},
				},
			},
		},
	}

	results := []xdr.OperationResult{
		{
			Code: xdr.OperationResultCodeOpInner,
			Tr: &xdr.OperationResultTr{
				Type: xdr.OperationTypeManageSellOffer,
				ManageSellOfferResult: &xdr.ManageSellOfferResult{
					Code: xdr.ManageSellOfferResultCodeManageSellOfferSuccess,
					Success: &xdr.ManageOfferSuccessResult{
						OffersClaimed: []xdr.ClaimAtom{offer1, offer2, offer3},
					},
				},
			},
		},
	}

	// Provide ledger entry changes so findTradeSellPrice can find the offer prices.
	unsafeMeta := xdr.TransactionMetaV1{
		Operations: []xdr.OperationMeta{
			{
				Changes: xdr.LedgerEntryChanges{
					// Offer 1 state+update
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
						State: &xdr.LedgerEntry{
							Data: xdr.LedgerEntryData{
								Type: xdr.LedgerEntryTypeOffer,
								Offer: &xdr.OfferEntry{
									SellerId: testAccount1ID,
									OfferId:  100,
									Price:    xdr.Price{N: 1000, D: 5000000},
								},
							},
						},
					},
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryRemoved,
						Removed: &xdr.LedgerKey{
							Type: xdr.LedgerEntryTypeOffer,
							Offer: &xdr.LedgerKeyOffer{
								SellerId: testAccount1ID,
								OfferId:  100,
							},
						},
					},
					// Offer 2 state+update
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
						State: &xdr.LedgerEntry{
							Data: xdr.LedgerEntryData{
								Type: xdr.LedgerEntryTypeOffer,
								Offer: &xdr.OfferEntry{
									SellerId: testAccount2ID,
									OfferId:  200,
									Price:    xdr.Price{N: 600, D: 3000000},
								},
							},
						},
					},
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryRemoved,
						Removed: &xdr.LedgerKey{
							Type: xdr.LedgerEntryTypeOffer,
							Offer: &xdr.LedgerKeyOffer{
								SellerId: testAccount2ID,
								OfferId:  200,
							},
						},
					},
					// Offer 3 state+update
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
						State: &xdr.LedgerEntry{
							Data: xdr.LedgerEntryData{
								Type: xdr.LedgerEntryTypeOffer,
								Offer: &xdr.OfferEntry{
									SellerId: testAccount3ID,
									OfferId:  300,
									Price:    xdr.Price{N: 400, D: 2000000},
								},
							},
						},
					},
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryRemoved,
						Removed: &xdr.LedgerKey{
							Type: xdr.LedgerEntryTypeOffer,
							Offer: &xdr.LedgerKeyOffer{
								SellerId: testAccount3ID,
								OfferId:  300,
							},
						},
					},
				},
			},
		},
	}

	tx := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result: wrapOperationsResultsSlice(results, true),
		UnsafeMeta: xdr.TransactionMeta{
			V:  1,
			V1: &unsafeMeta,
		},
	}

	// 2. Run the production transform — this is the same path export_trades uses.
	trades, err := TransformTrade(0, 100, tx, time.Unix(0, 0))
	if err != nil {
		t.Fatalf("TransformTrade returned unexpected error: %v", err)
	}

	// 3. Assert: one operation produced 3 trade rows.
	// GetTrades (internal/input/trades.go:50-73) counts this as 1 item
	// toward the --limit, but TransformTrade expands it into 3 rows.
	// With --limit 1, the CLI would emit all 3 rows — exceeding the limit.
	if len(trades) != 3 {
		t.Fatalf("expected 3 trade rows from one operation with 3 claimed offers, got %d", len(trades))
	}

	// GetTrades counts operations toward the limit: one ManageSellOffer = 1 toward limit.
	// This represents what GetTrades would return for limit=1: exactly 1 TradeTransformInput.
	operationCountTowardLimit := 1

	t.Logf("Operation count toward --limit: %d (GetTrades stops here for --limit 1)", operationCountTowardLimit)
	t.Logf("Actual trade rows produced:     %d (TransformTrade expands claimed offers)", len(trades))
	t.Logf("Mismatch: --limit 1 promised at most 1 trade row, but %d were produced", len(trades))

	if len(trades) <= operationCountTowardLimit {
		t.Errorf("expected trade rows (%d) to exceed the operation-level limit count (%d), "+
			"demonstrating the limit mismatch", len(trades), operationCountTowardLimit)
	}
}
```

### Test Output

```
=== RUN   TestTradesLimitCountsOperationsNotTradeRows
    data_integrity_poc_test.go:197: Operation count toward --limit: 1 (GetTrades stops here for --limit 1)
    data_integrity_poc_test.go:198: Actual trade rows produced:     3 (TransformTrade expands claimed offers)
    data_integrity_poc_test.go:199: Mismatch: --limit 1 promised at most 1 trade row, but 3 were produced
--- PASS: TestTradesLimitCountsOperationsNotTradeRows (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.735s
```
