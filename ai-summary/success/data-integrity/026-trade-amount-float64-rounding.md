# 026: history_trades amount columns round large claim atoms

**Date**: 2026-04-14
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformTrade()` exports the authoritative `history_trades.selling_amount` and `history_trades.buying_amount` columns by converting exact `ClaimAtom` `Int64` amounts into `float64`. Once the scaled trade amount is large enough, adjacent one-stroop values collapse to the same exported number.

This is reproducible on the normal trade path. For claim amounts `90071992547409930` and `90071992547409931`, `TransformTrade()` produces identical `SellingAmount` values and `encoding/json` serializes both rows with the same `selling_amount: 9007199254.740993`, so downstream consumers cannot recover which on-chain trade actually occurred.

## Root Cause

`TransformTrade()` keeps the exact claim-atom amounts until the final `TradeOutput` construction, then calls `utils.ConvertStroopValueToReal()` for both `selling_amount` and `buying_amount`. That helper converts the exact stroop integer through `big.Rat.Float64()`, rounding to the nearest IEEE-754 `float64` before JSON or Parquet serialization.

The trade schema exposes those rounded values as the primary monetary columns and does not carry an exact raw companion amount. Parquet repeats the same lossy representation by storing both fields as `DOUBLE`.

## Reproduction

Any normal export that includes a successful `ManageSellOffer`, `ManageBuyOffer`, `CreatePassiveSellOffer`, or path-payment trade can hit this path because `TransformTrade()` is the shared transformer for all of them. The underlying XDR offer claim amounts are ordinary `Int64` values, and the collision threshold is only about `9,007,199,254.740993` units, well below the protocol type's maximum range.

When two adjacent values above that threshold are transformed, the exported trade row contains the same rounded `selling_amount` / `buying_amount` float for both. Because no exact companion field is emitted, the one-stroop difference is permanently lost in both JSON and Parquet outputs.

## Affected Code

- `internal/transform/trade.go:TransformTrade:41-159` — reads exact claim amounts and writes `selling_amount` / `buying_amount` via `ConvertStroopValueToReal()`
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — rounds exact stroop integers to nearest `float64`
- `internal/transform/schema.go:TradeOutput:285-312` — exposes trade monetary columns only as `float64`
- `internal/transform/schema_parquet.go:TradeOutputParquet:218-244` — persists the same fields as Parquet `DOUBLE`
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:248-270` — forwards the rounded JSON values directly into Parquet output

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTradeAmountRoundsLargeClaimAtoms`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./... && go test ./internal/transform/... -run TestTradeAmountRoundsLargeClaimAtoms -count=1 -v` after creating the target file with the body below.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestTradeAmountRoundsLargeClaimAtoms(t *testing.T) {
	const amountA xdr.Int64 = 90071992547409930
	const amountB xdr.Int64 = 90071992547409931

	if amountA == amountB {
		t.Fatal("test setup error: amounts must differ by one stroop")
	}

	buildTx := func(soldAmount xdr.Int64) ingest.LedgerTransaction {
		envelope := xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: genericSourceAccount,
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

		offerEntry := xdr.LedgerEntry{
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: testAccount1ID,
					OfferId:  12345,
					Price:    xdr.Price{N: 1, D: 1},
				},
			},
		}

		return ingest.LedgerTransaction{
			Index: 1,
			Envelope: xdr.TransactionEnvelope{
				Type: xdr.EnvelopeTypeEnvelopeTypeTx,
				V1:   &envelope,
			},
			Result: wrapOperationsResultsSlice([]xdr.OperationResult{
				{
					Code: xdr.OperationResultCodeOpInner,
					Tr: &xdr.OperationResultTr{
						Type: xdr.OperationTypeManageSellOffer,
						ManageSellOfferResult: &xdr.ManageSellOfferResult{
							Code: xdr.ManageSellOfferResultCodeManageSellOfferSuccess,
							Success: &xdr.ManageOfferSuccessResult{
								OffersClaimed: []xdr.ClaimAtom{
									{
										Type: xdr.ClaimAtomTypeClaimAtomTypeOrderBook,
										OrderBook: &xdr.ClaimOfferAtom{
											SellerId:     testAccount1ID,
											OfferId:      12345,
											AssetSold:    nativeAsset,
											AssetBought:  usdtAsset,
											AmountSold:   soldAmount,
											AmountBought: 1000,
										},
									},
								},
							},
						},
					},
				},
			}, true),
			UnsafeMeta: xdr.TransactionMeta{
				V: 1,
				V1: &xdr.TransactionMetaV1{
					Operations: []xdr.OperationMeta{
						{
							Changes: xdr.LedgerEntryChanges{
								{
									Type:  xdr.LedgerEntryChangeTypeLedgerEntryState,
									State: &offerEntry,
								},
								{
									Type:    xdr.LedgerEntryChangeTypeLedgerEntryRemoved,
									Removed: &xdr.LedgerKey{Type: xdr.LedgerEntryTypeOffer},
								},
							},
						},
					},
				},
			},
		}
	}

	tradesA, err := TransformTrade(0, 100, buildTx(amountA), genericCloseTime)
	if err != nil {
		t.Fatalf("TransformTrade(amountA) returned error: %v", err)
	}
	if len(tradesA) != 1 {
		t.Fatalf("TransformTrade(amountA) returned %d trades, want 1", len(tradesA))
	}

	tradesB, err := TransformTrade(0, 100, buildTx(amountB), genericCloseTime)
	if err != nil {
		t.Fatalf("TransformTrade(amountB) returned error: %v", err)
	}
	if len(tradesB) != 1 {
		t.Fatalf("TransformTrade(amountB) returned %d trades, want 1", len(tradesB))
	}

	if tradesA[0].SellingAmount != tradesB[0].SellingAmount {
		t.Fatalf("expected rounded selling_amount collision, got %.20f and %.20f", tradesA[0].SellingAmount, tradesB[0].SellingAmount)
	}

	jsonA, err := json.Marshal(tradesA[0])
	if err != nil {
		t.Fatalf("marshal tradesA[0]: %v", err)
	}
	jsonB, err := json.Marshal(tradesB[0])
	if err != nil {
		t.Fatalf("marshal tradesB[0]: %v", err)
	}
	if string(jsonA) != string(jsonB) {
		t.Fatalf("expected serialized trade rows to collide, got\nA: %s\nB: %s", jsonA, jsonB)
	}

	if utils.ConvertStroopValueToReal(amountA) != utils.ConvertStroopValueToReal(amountB) {
		t.Fatalf("expected direct conversion collision for %d and %d", amountA, amountB)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Distinct on-chain trade claim amounts should remain distinguishable in exported monetary columns, e.g. `9007199254.7409930` vs `9007199254.7409931`.
- **Actual**: Both rows export the same rounded floating-point value and serialize as `selling_amount: 9007199254.740993`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC invokes the production `TransformTrade()` path with successful `ManageSellOffer` results and proves the serialized trade rows collide when only the claim amount changes.
2. Realistic preconditions: YES — successful offer claims are normal Stellar behavior, the source fields are ordinary `Int64` trade amounts, and the collision threshold is far below the type's maximum range.
3. Bug vs by-design: BUG — these are the canonical exported trade amount columns, not derived display-only helpers, and the schema provides no exact raw companion field that would let downstream systems recover the lost stroop.
4. Final severity: Critical — this silently corrupts monetary trade amounts in normal exports, so reconciliation, compliance, or analytics systems can consume plausible but wrong values.
5. In scope: YES — this is concrete financial data corruption in repository-owned ETL code.
6. Test correctness: CORRECT — the final test uses real production types, invokes the live transformer, shows the exact source values differ by one stroop, and confirms the collision survives JSON serialization.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Export trade amounts through an exact representation instead of making `float64` the only source of truth. The safest fix is to add exact raw or decimal-string trade amount columns and, if the rounded `float64` fields must remain for compatibility, treat them as derived convenience fields rather than the canonical monetary export.
