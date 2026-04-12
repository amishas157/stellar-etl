# H002: Sell-offer family detail amounts round large order sizes

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For successful `manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` operations, `history_operations.details.amount` should preserve the exact stroop-denominated order quantity. Two offers whose `BuyAmount`/`Amount` differ by 1 stroop should export different detail values rather than the same rounded `float64`.

## Mechanism

All three offer-family branches write `details["amount"]` via `utils.ConvertStroopValueToReal()`, which rounds large 7-decimal amounts to the nearest binary float. In the same file, the sibling formatter already uses `amount.String(...)` for these exact operation fields, and the live output map already mixes strings and numbers, so the precision loss is caused by this branch-local conversion choice rather than by a schema constraint.

## Trigger

1. Build successful `manage_buy_offer`, `manage_sell_offer`, or `create_passive_sell_offer` operations with `BuyAmount`/`Amount` set to `90071992547409930` stroops and `90071992547409931` stroops.
2. Run each through `TransformOperation()`.
3. Compare `OperationDetails["amount"]` across the two rows; the exact decimal strings differ, but the exported JSON number can collapse.

## Target Code

- `internal/transform/operation.go:701-755` — live offer-family detail export converts `amount` to `float64`
- `internal/transform/operation.go:1424-1455` — sibling formatter keeps exact `amount.String(...)` values for the same offer branches
- `internal/transform/schema.go:136-150` — operation details are emitted through `map[string]interface{}`

## Evidence

`manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` all populate `details["amount"]` from `utils.ConvertStroopValueToReal(...)`. The alternate formatter in the same file uses exact decimal strings for those same fields, which shows an exact representation is already available and consistent with this output surface.

## Anti-Evidence

The offer-family rows also emit exact `price_r` numerators and denominators, so price reconstruction still works. But there is no exact sibling for `amount`, so the requested order size itself is silently corrupted on the decoded operation-details surface.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — prior rejection (condensed fail 039) was based on "dead code" argument that was implicitly overturned by acceptance of findings 019 and 027, which use identical evidence patterns for different operation types

### Trace Summary

Traced the `extractOperationDetails()` function in `operation.go` for all three offer-family branches. Each branch stores `details["amount"]` via `utils.ConvertStroopValueToReal()`, which calls `big.NewRat(int64(input), 10000000).Float64()` — returning the nearest float64. For stroop values exceeding 2^53 (~9.007×10^15), distinct values collapse to the same float64. The `amount` package is already imported in the same file, and the sibling formatter at lines 1424-1455 demonstrates that `amount.String()` produces exact decimal strings for these same fields. The output container `OperationOutput.OperationDetails` is `map[string]interface{}`, which accepts strings without schema change.

### Code Paths Examined

- `internal/transform/operation.go:701-718` — `ManageBuyOffer` branch: `details["amount"] = utils.ConvertStroopValueToReal(op.BuyAmount)` — lossy float64 conversion
- `internal/transform/operation.go:720-737` — `ManageSellOffer` branch: `details["amount"] = utils.ConvertStroopValueToReal(op.Amount)` — lossy float64 conversion
- `internal/transform/operation.go:739-755` — `CreatePassiveSellOffer` branch: `details["amount"] = utils.ConvertStroopValueToReal(op.Amount)` — lossy float64 conversion
- `internal/transform/operation.go:1424-1434` — sibling `ManageBuyOffer` formatter: `details["amount"] = amount.String(op.BuyAmount)` — exact decimal string
- `internal/transform/operation.go:1435-1445` — sibling `ManageSellOffer` formatter: `details["amount"] = amount.String(op.Amount)` — exact decimal string
- `internal/transform/operation.go:1446-1455` — sibling `CreatePassiveSellOffer` formatter: `details["amount"] = amount.String(op.Amount)` — exact decimal string
- `internal/utils/main.go:84-88` — `ConvertStroopValueToReal()`: `big.NewRat(int64(input), 10000000).Float64()` — returns nearest float64
- `internal/transform/schema.go:142` — `OperationDetails map[string]interface{}` — can hold any type including strings

### Findings

The bug is real and follows the identical pattern as confirmed findings 019 (transfer-style ops) and 027 (path-payment ops). Three offer-family branches use `ConvertStroopValueToReal()` to produce a `float64` for `details["amount"]`, while the sibling formatter in the same file uses `amount.String()` for exact decimal representation. The `amount` package is already imported. The fix requires changing three lines — one per branch — from `utils.ConvertStroopValueToReal(...)` to `amount.String(...)`.

Note: A prior investigation (condensed fail entry 039) rejected this finding arguing that `transactionOperationWrapper.Details()` is "dead code" and therefore cannot serve as evidence of a better available path. However, findings 019 and 027 were subsequently accepted as VIABLE using that same evidence pattern. The acceptance of those findings establishes that the `map[string]interface{}` output surface's ability to hold exact strings — combined with the `amount` package being already imported — constitutes sufficient evidence that a better path exists.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or a new `internal/transform/data_integrity_poc_test.go` if one exists from prior PoCs)
- **Setup**: Construct two `ManageBuyOffer` operations with `BuyAmount` values `90071992547409930` and `90071992547409931` (differ by 1 stroop, exceed float64 precision). Use `utils.CreateSampleResultMeta(true, 1)` for result metadata.
- **Steps**: Call `TransformOperation()` on each operation. Extract `OperationDetails["amount"]` from both results.
- **Assertion**: Assert the two exported `amount` values are NOT equal. Currently they will be equal (both collapse to the same float64), demonstrating the precision loss. Repeat for `ManageSellOffer` and `CreatePassiveSellOffer` with `op.Amount`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestOfferDetailAmountRounding"
**Test Language**: Go

### Demonstration

The test constructs two operations for each of the three offer-family types (ManageBuyOffer, ManageSellOffer, CreatePassiveSellOffer) with BuyAmount/Amount values differing by 1 stroop (90071992547409930 vs 90071992547409931). After running through `TransformOperation()`, the exported `OperationDetails["amount"]` values are identical for both inputs in all three cases, proving that `ConvertStroopValueToReal()` silently corrupts large order sizes by collapsing distinct stroop values to the same float64.

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
					ScpValue: xdr.StellarValue{CloseTime: 0},
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
			t.Errorf("ManageBuyOffer amount precision loss: two distinct stroop values "+
				"(%d vs %d) produced the same exported amount: %v",
				stroopA, stroopB, amtA)
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
			t.Errorf("ManageSellOffer amount precision loss: two distinct stroop values "+
				"(%d vs %d) produced the same exported amount: %v",
				stroopA, stroopB, amtA)
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
			t.Errorf("CreatePassiveSellOffer amount precision loss: two distinct stroop values "+
				"(%d vs %d) produced the same exported amount: %v",
				stroopA, stroopB, amtA)
		}
	})
}
```

### Test Output

```
=== RUN   TestOfferDetailAmountRounding
=== RUN   TestOfferDetailAmountRounding/ManageBuyOffer
    data_integrity_poc_test.go:104: ManageBuyOffer amount precision loss: two distinct stroop values (90071992547409930 vs 90071992547409931) produced the same exported amount: 9.007199254740993e+09
=== RUN   TestOfferDetailAmountRounding/ManageSellOffer
    data_integrity_poc_test.go:154: ManageSellOffer amount precision loss: two distinct stroop values (90071992547409930 vs 90071992547409931) produced the same exported amount: 9.007199254740993e+09
=== RUN   TestOfferDetailAmountRounding/CreatePassiveSellOffer
    data_integrity_poc_test.go:202: CreatePassiveSellOffer amount precision loss: two distinct stroop values (90071992547409930 vs 90071992547409931) produced the same exported amount: 9.007199254740993e+09
--- FAIL: TestOfferDetailAmountRounding (0.00s)
    --- FAIL: TestOfferDetailAmountRounding/ManageBuyOffer (0.00s)
    --- FAIL: TestOfferDetailAmountRounding/ManageSellOffer (0.00s)
    --- FAIL: TestOfferDetailAmountRounding/CreatePassiveSellOffer (0.00s)
FAIL
```
