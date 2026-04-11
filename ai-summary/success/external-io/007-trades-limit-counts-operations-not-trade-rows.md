# 007: Trades Limit Counts Operations Not Trade Rows

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_trades --limit` applies its cap to `TradeTransformInput` items returned by `GetTrades()`, but each selected input can expand into multiple `TradeOutput` rows. As a result, the command can emit more trade rows than the CLI flag promises, and it can also stop on trade-capable operations that later transform into zero rows.

## Root Cause

`internal/input.GetTrades()` increments its limit counter once per successful trade-capable operation. `internal/transform.TransformTrade()` then expands a single operation into one row per claimed offer, and `cmd/export_trades` writes every returned row without any secondary limit enforcement.

## Reproduction

During normal operation, a manage-offer or path-payment transaction can cross multiple offers in the order book. When `export_trades` encounters such an operation within the first `N` capped inputs, it serializes all resulting trade rows even though the CLI help text says `--limit` is the maximum number of trades to export.

## Affected Code

- `internal/utils/main.go:AddArchiveFlags:250-254` — documents `--limit` as the maximum number of trades.
- `internal/input/trades.go:GetTrades:26-85` — caps successful trade-capable operations, not emitted trade rows.
- `internal/transform/trade.go:21-161` — expands one selected operation into `[]TradeOutput`, one row per claimed offer.
- `cmd/export_trades.go:28-58` — writes every transformed trade row without re-checking the limit.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTradesLimitCountsOperationsNotTradeRows`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"testing"
	"time"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestTradesLimitCountsOperationsNotTradeRows(t *testing.T) {
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

	unsafeMeta := xdr.TransactionMetaV1{
		Operations: []xdr.OperationMeta{
			{
				Changes: xdr.LedgerEntryChanges{
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

	trades, err := TransformTrade(0, 100, tx, time.Unix(0, 0))
	if err != nil {
		t.Fatalf("TransformTrade returned unexpected error: %v", err)
	}

	if len(trades) != 3 {
		t.Fatalf("expected 3 trade rows from one operation with 3 claimed offers, got %d", len(trades))
	}

	if len(trades) <= 1 {
		t.Fatalf("expected one trade-producing operation to expand beyond limit 1, got %d rows", len(trades))
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_trades --limit 1` should emit at most one trade row because the flag is documented as the maximum number of trades to export.
- **Actual**: the selected input operation can expand into multiple trade rows, and `export_trades` writes them all.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC proves one counted trade input expands into multiple emitted trade rows, and source tracing confirms `GetTrades()` applies the cap before that expansion.
2. Realistic preconditions: YES — multi-fill manage-offer and path-payment operations occur on mainnet order books; an independent live export over ledgers `28770265-28770365` contained operation `123567394517262340` emitted twice with different `order` values.
3. Bug vs by-design: BUG — the CLI help text explicitly promises a maximum number of trades, not a maximum number of trade-capable operations.
4. Final severity: Medium — the emitted rows are individually correct, but the command silently violates its documented export bound.
5. In scope: YES — this is a data-export correctness bug that can mislead downstream batch jobs and reconciliations.
6. Test correctness: CORRECT — the test uses the production trade transform path and real claimed-offer expansion, not mocks or tautological assertions.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Track the trade-row count after `TransformTrade()` in `cmd/export_trades`, or change `GetTrades()`/the command contract so the limit is explicitly defined in terms of trade-capable operations instead of emitted trade rows.
