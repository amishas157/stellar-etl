# H001: Path-payment detail amounts collapse distinct stroop values

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details` should preserve distinct path-payment amounts and bounds when the on-chain stroop values differ. For successful `path_payment_strict_receive` and `path_payment_strict_send` operations, fields such as `amount`, `source_max`, and `source_amount` should export different values for inputs like `90071992547409930` and `90071992547409931` stroops instead of collapsing them to the same JSON number.

## Mechanism

`extractOperationDetails()` converts the strict-receive `amount`, `source_max`, and successful `source_amount`, plus the strict-send `source_amount` and successful `amount`, through `utils.ConvertStroopValueToReal()`, which rounds to the nearest `float64`. The same `details` map already carries exact decimal strings for sibling fields such as `destination_min`, so this output surface can preserve exact decimal text; large path-payment amounts therefore silently lose one-stroop distinctions only because these branches pick the lossy conversion path.

## Trigger

1. Construct a successful `path_payment_strict_receive` with `DestAmount` or `SendMax` equal to `90071992547409930` stroops and another identical operation with `90071992547409931`.
2. Construct a successful `path_payment_strict_send` with `SendAmount` or resulting `DestAmount()` differing by 1 stroop at the same magnitude.
3. Run both through `TransformOperation()` and compare the exported `details.amount`, `details.source_max`, or `details.source_amount` values.

## Target Code

- `internal/transform/operation.go:619-699` — live path-payment detail export uses `utils.ConvertStroopValueToReal()` for path-payment amounts
- `internal/transform/operation.go:1379-1423` — sibling formatter keeps exact decimal strings for the same logical fields
- `internal/transform/schema.go:136-150` — operation details are exported as a heterogenous `map[string]interface{}`

## Evidence

The strict-receive branch stores `amount` and `source_max` as lossy `float64` values and overwrites `source_amount` with another lossy conversion on success. The strict-send branch similarly stores `source_amount` and successful `amount` as `float64`, even though the same file already uses `amount.String(...)` for `destination_min` and for the equivalent fields in the alternate formatter.

## Anti-Evidence

Downstream consumers can still recover exact values from raw XDR blobs elsewhere in the export, and `destination_min` is already exact on strict-send rows. But the `history_operations.details` payload itself is supposed to be a directly consumable decoded view, and these specific fields are wrong on that surface for large amounts.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (prior rejections 034-040 did not consider the live within-function `destination_min` inconsistency)

### Trace Summary

Traced `extractOperationDetails()` for both `PathPaymentStrictReceive` (lines 619-658) and `PathPaymentStrictSend` (lines 660-699) in `internal/transform/operation.go`. Confirmed that `amount`, `source_max`, `source_amount` (on success), and `source_amount`/`amount` (on success) all pass through `utils.ConvertStroopValueToReal()` which returns lossy `float64`. Critically, in the same function and same operation block, `destination_min` on strict-send (line 674) uses `amount.String(op.DestMin)` producing an exact decimal string. The structural analog `source_max` on strict-receive (line 633) uses the lossy float64 path. This within-function inconsistency in the live code distinguishes this finding from the rejected 034-040 group, which only cited dead code (`transactionOperationWrapper.Details()`) as evidence of a better path.

### Code Paths Examined

- `internal/transform/operation.go:619-658` — `PathPaymentStrictReceive`: `amount` (line 631), `source_max` (line 633), and successful `source_amount` (line 655) all use `utils.ConvertStroopValueToReal()` returning lossy float64
- `internal/transform/operation.go:660-699` — `PathPaymentStrictSend`: `source_amount` (line 673) uses `utils.ConvertStroopValueToReal()` (lossy float64), `destination_min` (line 674) uses `amount.String()` (exact string), successful `amount` (line 696) uses `utils.ConvertStroopValueToReal()` (lossy float64)
- `internal/transform/operation.go:1379-1423` — `transactionOperationWrapper.Details()` uses `amount.String()` for all path-payment fields; confirmed dead code (zero callers outside tests)
- `internal/utils/main.go:85-88` — `ConvertStroopValueToReal()` uses `big.NewRat(int64(input), 10000000).Float64()`, the best possible float64 conversion but still lossy for values exceeding float64 precision at 7-decimal scale
- `internal/transform/schema.go:136-150` — `OperationOutput.OperationDetails` is `map[string]interface{}`, capable of holding either strings or floats

### Findings

The `destination_min` field on `PathPaymentStrictSend` already uses `amount.String()` in the same live function (`extractOperationDetails()`), same details map, same operation-type block. This proves the map's consumers already handle string-valued amount fields. The use of `ConvertStroopValueToReal()` for `source_amount`, `amount`, and `source_max` is therefore an oversight, not a schema constraint.

Prior rejections (034-040) covered "path-payment settlement amounts" but only cited the dead-code `transactionOperationWrapper.Details()` as evidence of a better path. The fail summary's own framework distinguishes: (a) "a better path exists but is bypassed → VIABLE" vs (c) "goes directly to float64 via best conversion → NOT_VIABLE schema limit." The live `destination_min` inconsistency satisfies criterion (a).

This finding is also analogous to success #019 (transfer-style operation details float64 rounding for `create_account`, `payment`, etc.) but covers the distinct path-payment operation types not included in that finding.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or a new `internal/transform/data_integrity_poc_test.go`)
- **Setup**: Build two `PathPaymentStrictSend` operations with `SendAmount` values of `90071992547409930` and `90071992547409931` stroops. Construct successful transaction results with `DestAmount()` values differing by 1 stroop at the same magnitude.
- **Steps**: Call `TransformOperation()` for each, extract `OperationDetails["source_amount"]` and `OperationDetails["amount"]`.
- **Assertion**: Assert that the two `source_amount` float64 values are equal (demonstrating the collision), and compare against the exact `amount.String()` representations which remain distinct. Also verify that `OperationDetails["destination_min"]` IS a string (demonstrating the within-function inconsistency).

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestPathPaymentDetailAmountRounding"
**Test Language**: Go

### Demonstration

The test constructs two `PathPaymentStrictSend` and two `PathPaymentStrictReceive` operations whose stroop values differ by 1 (90071992547409930 vs 90071992547409931). After running through `TransformOperation()`, the `source_amount` and `amount` fields (which use `ConvertStroopValueToReal()`) collapse both inputs to the identical float64 value `9.007199254740993e+09`, while `destination_min` (which uses `amount.String()`) correctly preserves the distinction as `"9007199254.7409930"` vs `"9007199254.7409931"`. This proves the within-function inconsistency: the same details map uses a lossy float64 path for some fields and an exact string path for others.

### Test Body

```go
package transform

import (
	"fmt"
	"testing"

	"github.com/stellar/go-stellar-sdk/amount"
	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestPathPaymentDetailAmountRounding demonstrates that path-payment detail
// amounts exported via ConvertStroopValueToReal() collapse distinct stroop
// values to the same float64, while the sibling destination_min field (which
// uses amount.String()) preserves exact values.
func TestPathPaymentDetailAmountRounding(t *testing.T) {
	// Two stroop values that differ by 1 but map to the same float64
	stroopA := xdr.Int64(90071992547409930)
	stroopB := xdr.Int64(90071992547409931)

	// Sanity: the exact string representations MUST differ
	strA := amount.String(stroopA)
	strB := amount.String(stroopB)
	if strA == strB {
		t.Fatalf("precondition failed: amount.String values should differ, got %s for both", strA)
	}

	// Sanity: the lossy float64 representations MUST collide
	floatA := utils.ConvertStroopValueToReal(stroopA)
	floatB := utils.ConvertStroopValueToReal(stroopB)
	if floatA != floatB {
		t.Fatalf("precondition failed: float64 values should be equal (collision), got %v vs %v", floatA, floatB)
	}

	// --- PathPaymentStrictSend: source_amount is lossy, destination_min is exact ---

	buildStrictSendTx := func(sendAmount, destMin, resultDestAmount xdr.Int64) (xdr.Operation, ingest.LedgerTransaction) {
		op := xdr.Operation{
			SourceAccount: &testAccount3,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypePathPaymentStrictSend,
				PathPaymentStrictSendOp: &xdr.PathPaymentStrictSendOp{
					SendAsset:   nativeAsset,
					SendAmount:  sendAmount,
					Destination: testAccount4,
					DestAsset:   nativeAsset,
					DestMin:     destMin,
					Path:        []xdr.Asset{},
				},
			},
		}

		envelope := xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: testAccount3,
				Operations:    []xdr.Operation{op},
				Ext: xdr.TransactionExt{
					V:           0,
					SorobanData: &xdr.SorobanTransactionData{Ext: xdr.SorobanTransactionDataExt{V: 0}, Resources: xdr.SorobanResources{Footprint: xdr.LedgerFootprint{ReadOnly: []xdr.LedgerKey{}, ReadWrite: []xdr.LedgerKey{}}}, ResourceFee: 0},
				},
			},
		}

		opResults := []xdr.OperationResult{
			{
				Code: xdr.OperationResultCodeOpInner,
				Tr: &xdr.OperationResultTr{
					Type: xdr.OperationTypePathPaymentStrictSend,
					PathPaymentStrictSendResult: &xdr.PathPaymentStrictSendResult{
						Code: xdr.PathPaymentStrictSendResultCodePathPaymentStrictSendSuccess,
						Success: &xdr.PathPaymentStrictSendResultSuccess{
							Last: xdr.SimplePaymentResult{Amount: resultDestAmount},
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
			Result: xdr.TransactionResultPair{
				Result: xdr.TransactionResult{
					Result: xdr.TransactionResultResult{
						Code:    xdr.TransactionResultCodeTxSuccess,
						Results: &opResults,
					},
				},
			},
			UnsafeMeta: xdr.TransactionMeta{
				V: 1,
				V1: &xdr.TransactionMetaV1{
					Operations: []xdr.OperationMeta{{}},
				},
			},
		}
		return op, tx
	}

	lcm := makeLedgerCloseMeta()

	// Build two strict-send operations whose SendAmount differs by 1 stroop
	opA, txA := buildStrictSendTx(stroopA, stroopA, stroopA)
	opB, txB := buildStrictSendTx(stroopB, stroopB, stroopB)

	outA, err := TransformOperation(opA, 0, txA, 1, lcm, "")
	if err != nil {
		t.Fatalf("TransformOperation A failed: %v", err)
	}
	outB, err := TransformOperation(opB, 0, txB, 1, lcm, "")
	if err != nil {
		t.Fatalf("TransformOperation B failed: %v", err)
	}

	// 1. source_amount (lossy float64) — should be equal despite different inputs
	srcAmtA := outA.OperationDetails["source_amount"]
	srcAmtB := outB.OperationDetails["source_amount"]
	srcAmtAFloat, okA := srcAmtA.(float64)
	srcAmtBFloat, okB := srcAmtB.(float64)
	if !okA || !okB {
		t.Fatalf("source_amount not float64: A=%T, B=%T", srcAmtA, srcAmtB)
	}
	if srcAmtAFloat != srcAmtBFloat {
		t.Errorf("Expected source_amount collision but got different values: %v vs %v", srcAmtAFloat, srcAmtBFloat)
	} else {
		t.Logf("BUG CONFIRMED: source_amount collapsed two distinct stroops (%d, %d) to same float64 %v",
			stroopA, stroopB, srcAmtAFloat)
	}

	// 2. amount (success path, lossy float64) — should also be equal despite different inputs
	amtA := outA.OperationDetails["amount"]
	amtB := outB.OperationDetails["amount"]
	amtAFloat, okA := amtA.(float64)
	amtBFloat, okB := amtB.(float64)
	if !okA || !okB {
		t.Fatalf("amount not float64: A=%T, B=%T", amtA, amtB)
	}
	if amtAFloat != amtBFloat {
		t.Errorf("Expected amount collision but got different values: %v vs %v", amtAFloat, amtBFloat)
	} else {
		t.Logf("BUG CONFIRMED: amount (success path) collapsed two distinct stroops (%d, %d) to same float64 %v",
			stroopA, stroopB, amtAFloat)
	}

	// 3. destination_min (exact string via amount.String) — should be different
	destMinA := outA.OperationDetails["destination_min"]
	destMinB := outB.OperationDetails["destination_min"]
	destMinAStr, okA := destMinA.(string)
	destMinBStr, okB := destMinB.(string)
	if !okA || !okB {
		t.Fatalf("destination_min not string: A=%T, B=%T", destMinA, destMinB)
	}
	if destMinAStr == destMinBStr {
		t.Errorf("destination_min should be distinct but both are %q", destMinAStr)
	} else {
		t.Logf("destination_min correctly preserves distinct values: %q vs %q", destMinAStr, destMinBStr)
	}

	// 4. Within-function inconsistency: source_amount is float64, destination_min is string
	t.Logf("Within-function type inconsistency: source_amount is %T, destination_min is %T",
		srcAmtA, destMinA)

	// Summary: path-payment details lose precision for source_amount and amount
	// while destination_min in the same function preserves it via amount.String()
	t.Logf("Summary: %d and %d stroops both export source_amount=%.10f and amount=%.10f",
		stroopA, stroopB, srcAmtAFloat, amtAFloat)
	t.Logf("But destination_min correctly distinguishes: %q vs %q", destMinAStr, destMinBStr)

	// --- PathPaymentStrictReceive: amount, source_max, and source_amount are all lossy ---

	buildStrictReceiveTx := func(destAmount, sendMax, resultSendAmount xdr.Int64) (xdr.Operation, ingest.LedgerTransaction) {
		op := xdr.Operation{
			SourceAccount: &testAccount3,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypePathPaymentStrictReceive,
				PathPaymentStrictReceiveOp: &xdr.PathPaymentStrictReceiveOp{
					SendAsset:   nativeAsset,
					SendMax:     sendMax,
					Destination: testAccount4,
					DestAsset:   nativeAsset,
					DestAmount:  destAmount,
					Path:        []xdr.Asset{},
				},
			},
		}

		envelope := xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: testAccount3,
				Operations:    []xdr.Operation{op},
				Ext: xdr.TransactionExt{
					V:           0,
					SorobanData: &xdr.SorobanTransactionData{Ext: xdr.SorobanTransactionDataExt{V: 0}, Resources: xdr.SorobanResources{Footprint: xdr.LedgerFootprint{ReadOnly: []xdr.LedgerKey{}, ReadWrite: []xdr.LedgerKey{}}}, ResourceFee: 0},
				},
			},
		}

		opResults := []xdr.OperationResult{
			{
				Code: xdr.OperationResultCodeOpInner,
				Tr: &xdr.OperationResultTr{
					Type: xdr.OperationTypePathPaymentStrictReceive,
					PathPaymentStrictReceiveResult: &xdr.PathPaymentStrictReceiveResult{
						Code: xdr.PathPaymentStrictReceiveResultCodePathPaymentStrictReceiveSuccess,
						Success: &xdr.PathPaymentStrictReceiveResultSuccess{
							Last: xdr.SimplePaymentResult{Amount: resultSendAmount},
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
			Result: xdr.TransactionResultPair{
				Result: xdr.TransactionResult{
					Result: xdr.TransactionResultResult{
						Code:    xdr.TransactionResultCodeTxSuccess,
						Results: &opResults,
					},
				},
			},
			UnsafeMeta: xdr.TransactionMeta{
				V: 1,
				V1: &xdr.TransactionMetaV1{
					Operations: []xdr.OperationMeta{{}},
				},
			},
		}
		return op, tx
	}

	// Strict-receive: amount, source_max, and source_amount are all lossy
	opRA, txRA := buildStrictReceiveTx(stroopA, stroopA, stroopA)
	opRB, txRB := buildStrictReceiveTx(stroopB, stroopB, stroopB)

	outRA, err := TransformOperation(opRA, 0, txRA, 1, lcm, "")
	if err != nil {
		t.Fatalf("TransformOperation strict-receive A failed: %v", err)
	}
	outRB, err := TransformOperation(opRB, 0, txRB, 1, lcm, "")
	if err != nil {
		t.Fatalf("TransformOperation strict-receive B failed: %v", err)
	}

	// Check all three lossy fields
	for _, field := range []string{"amount", "source_max", "source_amount"} {
		vA := outRA.OperationDetails[field]
		vB := outRB.OperationDetails[field]
		fA, okA := vA.(float64)
		fB, okB := vB.(float64)
		if !okA || !okB {
			t.Fatalf("strict-receive %s not float64: A=%T, B=%T", field, vA, vB)
		}
		if fA != fB {
			t.Errorf("Expected strict-receive %s collision but got different values: %v vs %v", field, fA, fB)
		} else {
			t.Logf("BUG CONFIRMED (strict-receive): %s collapsed stroops %d and %d to same float64 %v",
				field, stroopA, stroopB, fA)
		}
	}

	// Print expected exact strings for reference
	t.Logf("Exact string representations: %s vs %s", amount.String(stroopA), amount.String(stroopB))
	fmt.Println("--- PoC complete: path-payment detail amount rounding confirmed ---")
}
```

### Test Output

```
=== RUN   TestPathPaymentDetailAmountRounding
    data_integrity_poc_test.go:131: BUG CONFIRMED: source_amount collapsed two distinct stroops (90071992547409930, 90071992547409931) to same float64 9.007199254740993e+09
    data_integrity_poc_test.go:146: BUG CONFIRMED: amount (success path) collapsed two distinct stroops (90071992547409930, 90071992547409931) to same float64 9.007199254740993e+09
    data_integrity_poc_test.go:161: destination_min correctly preserves distinct values: "9007199254.7409930" vs "9007199254.7409931"
    data_integrity_poc_test.go:165: Within-function type inconsistency: source_amount is float64, destination_min is string
    data_integrity_poc_test.go:170: Summary: 90071992547409930 and 90071992547409931 stroops both export source_amount=9007199254.7409934998 and amount=9007199254.7409934998
    data_integrity_poc_test.go:172: But destination_min correctly distinguishes: "9007199254.7409930" vs "9007199254.7409931"
    data_integrity_poc_test.go:267: BUG CONFIRMED (strict-receive): amount collapsed stroops 90071992547409930 and 90071992547409931 to same float64 9.007199254740993e+09
    data_integrity_poc_test.go:267: BUG CONFIRMED (strict-receive): source_max collapsed stroops 90071992547409930 and 90071992547409931 to same float64 9.007199254740993e+09
    data_integrity_poc_test.go:267: BUG CONFIRMED (strict-receive): source_amount collapsed stroops 90071992547409930 and 90071992547409931 to same float64 9.007199254740993e+09
    data_integrity_poc_test.go:273: Exact string representations: 9007199254.7409930 vs 9007199254.7409931
--- PoC complete: path-payment detail amount rounding confirmed ---
--- PASS: TestPathPaymentDetailAmountRounding (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.764s
```
