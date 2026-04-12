# 028: Offer-detail amount rounding

**Date**: 2026-04-12
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports `manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` quantities in `history_operations.details.amount` as `float64`. Once the on-chain stroop value is large enough, adjacent valid offer amounts collapse to the same JSON number even though the same output surface already carries exact decimal strings for other monetary fields.

## Root Cause

`extractOperationDetails()` routes these three offer-family `amount` fields through `utils.ConvertStroopValueToReal()`, which scales the `xdr.Int64` stroop value and returns the nearest `float64`. At magnitudes above float64's exact 7-decimal precision, the low-order stroops are rounded away. This is not required by the output schema: `OperationOutput.OperationDetails` is a `map[string]interface{}`, and the same live function already stores exact decimal strings such as `destination_min` with `amount.String(...)`.

## Reproduction

Create two otherwise identical successful offer operations whose `BuyAmount`/`Amount` differ by 1 stroop: `90071992547409930` and `90071992547409931`. Run each through `TransformOperation()` and compare `OperationDetails["amount"]`; for all three offer-family types, the exact decimal values differ but the exported `float64` is the same.

## Affected Code

- `internal/transform/operation.go:extractOperationDetails:701-745` — live offer-family detail export converts `BuyAmount`/`Amount` through `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:extractOperationDetails:660-675` — the same live details map already carries exact decimal strings via `amount.String(...)`
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — converts stroops to the nearest `float64`, which cannot preserve all adjacent 7-decimal amounts at large magnitudes
- `internal/transform/schema.go:OperationOutput:136-150` — operation details are exported through `map[string]interface{}`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestOfferDetailAmountRounding`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./internal/transform/... -run TestOfferDetailAmountRounding -v`.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestOfferDetailAmountRounding demonstrates that offer-family operations lose
// precision when exporting the "amount" detail field via ConvertStroopValueToReal.
// Two amounts differing by 1 stroop (beyond float64 precision) collapse to the
// same exported value.
func TestOfferDetailAmountRounding(t *testing.T) {
	// Two stroop values that differ by 1 but exceed float64 53-bit mantissa precision.
	stroopA := xdr.Int64(90071992547409930)
	stroopB := xdr.Int64(90071992547409931)

	lcm := xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					ScpValue:  xdr.StellarValue{CloseTime: 0},
					LedgerSeq: 1,
				},
			},
		},
	}

	buildTx := func(ops []xdr.Operation) ingest.LedgerTransaction {
		envelope := xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: genericSourceAccount,
				Memo:          xdr.Memo{},
				Operations:    ops,
				Ext: xdr.TransactionExt{
					V: 0,
					SorobanData: &xdr.SorobanTransactionData{
						Ext:         xdr.SorobanTransactionDataExt{V: 0},
						Resources:   xdr.SorobanResources{Footprint: xdr.LedgerFootprint{}},
						ResourceFee: 100,
					},
				},
			},
		}
		return ingest.LedgerTransaction{
			Index: 1,
			Envelope: xdr.TransactionEnvelope{
				Type: xdr.EnvelopeTypeEnvelopeTypeTx,
				V1:   &envelope,
			},
			Result:     utils.CreateSampleResultMeta(true, len(ops)).Result,
			UnsafeMeta: xdr.TransactionMeta{V: 1, V1: genericTxMeta},
		}
	}

	t.Run("ManageBuyOffer", func(t *testing.T) {
		opA := xdr.Operation{
			SourceAccount: &genericSourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeManageBuyOffer,
				ManageBuyOfferOp: &xdr.ManageBuyOfferOp{
					Buying:    nativeAsset,
					Selling:   usdtAsset,
					BuyAmount: stroopA,
					Price:     xdr.Price{N: 1, D: 1},
					OfferId:   0,
				},
			},
		}
		opB := xdr.Operation{
			SourceAccount: &genericSourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeManageBuyOffer,
				ManageBuyOfferOp: &xdr.ManageBuyOfferOp{
					Buying:    nativeAsset,
					Selling:   usdtAsset,
					BuyAmount: stroopB,
					Price:     xdr.Price{N: 1, D: 1},
					OfferId:   0,
				},
			},
		}

		txA := buildTx([]xdr.Operation{opA})
		txB := buildTx([]xdr.Operation{opB})

		outA, err := TransformOperation(opA, 0, txA, 1, lcm, "")
		if err != nil {
			t.Fatalf("TransformOperation A: %v", err)
		}
		outB, err := TransformOperation(opB, 0, txB, 1, lcm, "")
		if err != nil {
			t.Fatalf("TransformOperation B: %v", err)
		}

		amtA := outA.OperationDetails["amount"]
		amtB := outB.OperationDetails["amount"]

		if amtA == amtB {
			t.Errorf("ManageBuyOffer amount precision loss: two distinct stroop values (%d vs %d) produced the same exported amount: %v", stroopA, stroopB, amtA)
		}
	})

	t.Run("ManageSellOffer", func(t *testing.T) {
		opA := xdr.Operation{
			SourceAccount: &genericSourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeManageSellOffer,
				ManageSellOfferOp: &xdr.ManageSellOfferOp{
					Selling: usdtAsset,
					Buying:  nativeAsset,
					Amount:  stroopA,
					Price:   xdr.Price{N: 1, D: 1},
					OfferId: 0,
				},
			},
		}
		opB := xdr.Operation{
			SourceAccount: &genericSourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeManageSellOffer,
				ManageSellOfferOp: &xdr.ManageSellOfferOp{
					Selling: usdtAsset,
					Buying:  nativeAsset,
					Amount:  stroopB,
					Price:   xdr.Price{N: 1, D: 1},
					OfferId: 0,
				},
			},
		}

		txA := buildTx([]xdr.Operation{opA})
		txB := buildTx([]xdr.Operation{opB})

		outA, err := TransformOperation(opA, 0, txA, 1, lcm, "")
		if err != nil {
			t.Fatalf("TransformOperation A: %v", err)
		}
		outB, err := TransformOperation(opB, 0, txB, 1, lcm, "")
		if err != nil {
			t.Fatalf("TransformOperation B: %v", err)
		}

		amtA := outA.OperationDetails["amount"]
		amtB := outB.OperationDetails["amount"]

		if amtA == amtB {
			t.Errorf("ManageSellOffer amount precision loss: two distinct stroop values (%d vs %d) produced the same exported amount: %v", stroopA, stroopB, amtA)
		}
	})

	t.Run("CreatePassiveSellOffer", func(t *testing.T) {
		opA := xdr.Operation{
			SourceAccount: &genericSourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeCreatePassiveSellOffer,
				CreatePassiveSellOfferOp: &xdr.CreatePassiveSellOfferOp{
					Selling: usdtAsset,
					Buying:  nativeAsset,
					Amount:  stroopA,
					Price:   xdr.Price{N: 1, D: 1},
				},
			},
		}
		opB := xdr.Operation{
			SourceAccount: &genericSourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeCreatePassiveSellOffer,
				CreatePassiveSellOfferOp: &xdr.CreatePassiveSellOfferOp{
					Selling: usdtAsset,
					Buying:  nativeAsset,
					Amount:  stroopB,
					Price:   xdr.Price{N: 1, D: 1},
				},
			},
		}

		txA := buildTx([]xdr.Operation{opA})
		txB := buildTx([]xdr.Operation{opB})

		outA, err := TransformOperation(opA, 0, txA, 1, lcm, "")
		if err != nil {
			t.Fatalf("TransformOperation A: %v", err)
		}
		outB, err := TransformOperation(opB, 0, txB, 1, lcm, "")
		if err != nil {
			t.Fatalf("TransformOperation B: %v", err)
		}

		amtA := outA.OperationDetails["amount"]
		amtB := outB.OperationDetails["amount"]

		if amtA == amtB {
			t.Errorf("CreatePassiveSellOffer amount precision loss: two distinct stroop values (%d vs %d) produced the same exported amount: %v", stroopA, stroopB, amtA)
		}
	})
}
```

## Expected vs Actual Behavior

- **Expected**: offer-operation detail amounts should preserve the exact stroop-denominated order quantity, or at minimum keep adjacent 1-stroop values distinguishable.
- **Actual**: large `BuyAmount`/`Amount` values round to the same `float64`, so distinct on-chain offer quantities export as the same plausible JSON number.

## Adversarial Review

1. Exercises claimed bug: YES — the test constructs real `xdr.Operation` values for all three offer-family types and runs the production `TransformOperation()` path.
2. Realistic preconditions: YES — these operation fields are protocol-level `xdr.Int64` amounts, and values above float64's exact 7-decimal range are valid on-chain offer quantities.
3. Bug vs by-design: BUG — the live operation-details map already mixes strings and numbers for monetary fields, so exporting these three amounts as lossy `float64` is an inconsistent conversion choice rather than a schema requirement.
4. Final severity: Critical — this silently corrupts exported monetary order sizes in `history_operations.details`.
5. In scope: YES — it is a concrete data-correctness defect in `internal/transform/`.
6. Test correctness: CORRECT — two distinct inputs that differ by exactly 1 stroop should remain distinguishable, and the test shows the production export makes them equal.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Replace the three offer-family `details["amount"]` assignments with exact decimal formatting such as `amount.String(op.BuyAmount)` / `amount.String(op.Amount)`, consistent with the existing live `map[string]interface{}` output surface.
