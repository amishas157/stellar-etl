# 027: Path-payment operation details round large amounts

**Date**: 2026-04-12
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports several path-payment amount fields in `history_operations.details` as `float64`, so distinct large stroop values can collapse to the same JSON number. In both `path_payment_strict_receive` and `path_payment_strict_send`, adjacent on-chain values that differ by 1 stroop become indistinguishable even though the same live details map already preserves sibling monetary data as exact decimal strings.

## Root Cause

`extractOperationDetails()` sends `amount`, `source_max`, and `source_amount` for the path-payment branches through `utils.ConvertStroopValueToReal()`, which returns the nearest `float64` to the scaled decimal value. Once the value exceeds float64 precision at 7-decimal stroop scale, low-order stroops are rounded away. This loss is avoidable on this output surface because `OperationOutput.OperationDetails` is a `map[string]interface{}` and the same live branch already stores `destination_min` as an exact `amount.String(...)` value.

## Reproduction

Create two otherwise identical successful path-payment operations whose stroop amounts differ by 1: `90071992547409930` and `90071992547409931`. Run both through `TransformOperation()` and compare the exported `details.source_amount`, `details.amount`, and `details.source_max` values: the production code emits the same `float64` for both rows, while `destination_min` still distinguishes the exact decimal strings `9007199254.7409930` and `9007199254.7409931`.

## Affected Code

- `internal/transform/operation.go:extractOperationDetails:619-699` — live path-payment detail export converts affected amount fields through `utils.ConvertStroopValueToReal()`
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — converts stroops to nearest `float64`, which cannot preserve all 7-decimal stroop distinctions at large magnitudes
- `internal/transform/schema.go:OperationOutput:136-150` — operation details are exported through `map[string]interface{}`, which can already carry exact string values

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPathPaymentDetailAmountRounding`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./internal/transform/... -run TestPathPaymentDetailAmountRounding -v`.

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

func TestPathPaymentDetailAmountRounding(t *testing.T) {
	stroopA := xdr.Int64(90071992547409930)
	stroopB := xdr.Int64(90071992547409931)

	strA := amount.String(stroopA)
	strB := amount.String(stroopB)
	if strA == strB {
		t.Fatalf("precondition failed: amount.String values should differ, got %s for both", strA)
	}

	floatA := utils.ConvertStroopValueToReal(stroopA)
	floatB := utils.ConvertStroopValueToReal(stroopB)
	if floatA != floatB {
		t.Fatalf("precondition failed: float64 values should be equal, got %v vs %v", floatA, floatB)
	}

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
					V: 0,
					SorobanData: &xdr.SorobanTransactionData{
						Ext: xdr.SorobanTransactionDataExt{V: 0},
						Resources: xdr.SorobanResources{
							Footprint: xdr.LedgerFootprint{
								ReadOnly:  []xdr.LedgerKey{},
								ReadWrite: []xdr.LedgerKey{},
							},
						},
						ResourceFee: 0,
					},
				},
			},
		}

		opResults := []xdr.OperationResult{{
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
		}}

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
					V: 0,
					SorobanData: &xdr.SorobanTransactionData{
						Ext: xdr.SorobanTransactionDataExt{V: 0},
						Resources: xdr.SorobanResources{
							Footprint: xdr.LedgerFootprint{
								ReadOnly:  []xdr.LedgerKey{},
								ReadWrite: []xdr.LedgerKey{},
							},
						},
						ResourceFee: 0,
					},
				},
			},
		}

		opResults := []xdr.OperationResult{{
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
		}}

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

	opA, txA := buildStrictSendTx(stroopA, stroopA, stroopA)
	opB, txB := buildStrictSendTx(stroopB, stroopB, stroopB)

	outA, err := TransformOperation(opA, 0, txA, 1, lcm, "")
	if err != nil {
		t.Fatalf("TransformOperation strict-send A failed: %v", err)
	}
	outB, err := TransformOperation(opB, 0, txB, 1, lcm, "")
	if err != nil {
		t.Fatalf("TransformOperation strict-send B failed: %v", err)
	}

	srcAmtA, okA := outA.OperationDetails["source_amount"].(float64)
	srcAmtB, okB := outB.OperationDetails["source_amount"].(float64)
	if !okA || !okB {
		t.Fatalf("strict-send source_amount not float64: A=%T, B=%T", outA.OperationDetails["source_amount"], outB.OperationDetails["source_amount"])
	}
	if srcAmtA != srcAmtB {
		t.Fatalf("expected strict-send source_amount collision, got %v vs %v", srcAmtA, srcAmtB)
	}

	amtA, okA := outA.OperationDetails["amount"].(float64)
	amtB, okB := outB.OperationDetails["amount"].(float64)
	if !okA || !okB {
		t.Fatalf("strict-send amount not float64: A=%T, B=%T", outA.OperationDetails["amount"], outB.OperationDetails["amount"])
	}
	if amtA != amtB {
		t.Fatalf("expected strict-send amount collision, got %v vs %v", amtA, amtB)
	}

	destMinA, okA := outA.OperationDetails["destination_min"].(string)
	destMinB, okB := outB.OperationDetails["destination_min"].(string)
	if !okA || !okB {
		t.Fatalf("strict-send destination_min not string: A=%T, B=%T", outA.OperationDetails["destination_min"], outB.OperationDetails["destination_min"])
	}
	if destMinA == destMinB {
		t.Fatalf("expected strict-send destination_min to remain distinct, both were %q", destMinA)
	}

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

	for _, field := range []string{"amount", "source_max", "source_amount"} {
		fA, okA := outRA.OperationDetails[field].(float64)
		fB, okB := outRB.OperationDetails[field].(float64)
		if !okA || !okB {
			t.Fatalf("strict-receive %s not float64: A=%T, B=%T", field, outRA.OperationDetails[field], outRB.OperationDetails[field])
		}
		if fA != fB {
			t.Fatalf("expected strict-receive %s collision, got %v vs %v", field, fA, fB)
		}
	}

	t.Logf("strict-send source_amount: %v == %v", srcAmtA, srcAmtB)
	t.Logf("strict-send amount: %v == %v", amtA, amtB)
	t.Logf("strict-send destination_min: %q != %q", destMinA, destMinB)
	t.Logf("exact string representations: %s != %s", strA, strB)
	fmt.Println("--- PoC complete: path-payment detail amount rounding confirmed ---")
}
```

## Expected vs Actual Behavior

- **Expected**: path-payment detail exports should preserve distinct stroop values, either by exporting exact decimal strings or by otherwise keeping adjacent 7-decimal amounts distinguishable.
- **Actual**: large `amount`, `source_max`, and `source_amount` values collapse to the same `float64`, so different on-chain monetary values export as the same plausible JSON number.

## Adversarial Review

1. Exercises claimed bug: YES — the test builds real `xdr.Operation` values for both path-payment variants, runs the production `TransformOperation()` path, and compares the exported detail fields.
2. Realistic preconditions: YES — these amount fields are ordinary `xdr.Int64` stroop amounts, and the trigger value is below Stellar’s representable asset and lumen ranges.
3. Bug vs by-design: BUG — this output surface already permits exact decimal strings in the same live branch (`destination_min`), so the precision loss is a local conversion choice rather than an unavoidable schema constraint.
4. Final severity: Critical — the bug silently corrupts exported monetary values in `history_operations.details`, a directly consumed analytics surface.
5. In scope: YES — it is a concrete data-correctness issue in `internal/transform/`.
6. Test correctness: CORRECT — the assertions compare exact decimal-string distinctions against the actual production export values and prove the collision happens in live code, not in test scaffolding.
7. Alternative explanations: NONE — the two input stroop values are genuinely distinct, and the collision only appears after the production float conversion.
8. Novelty: NOVEL

## Suggested Fix

Stop exporting these path-payment detail amounts as `float64`. Use exact decimal strings such as `amount.String(...)` for `amount`, `source_max`, and `source_amount`, consistent with the existing `destination_min` field and the exact-string formatter elsewhere in `operation.go`.
