# 021: Path-payment strict-send amount type flip

**Date**: 2026-04-12
**Severity**: High
**Impact**: structural data corruption
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`history_operations.details.amount` for `path_payment_strict_send` does not keep a stable JSON type. Failed rows export `"amount":"0.0000000"` while successful rows export `"amount":0.2`, so downstream consumers see schema drift for the same field name within the same operation type.

## Root Cause

`extractOperationDetails()` initializes strict-send `details["amount"]` with `amount.String(0)`, which is a Go `string`, and only overwrites it with `utils.ConvertStroopValueToReal(result.DestAmount())`, which is a `float64`, when `transaction.Result.Successful()` is true. `TransformOperation()` then stores that mixed-typed `map[string]interface{}` directly in `OperationOutput`, so JSON serialization preserves the branch-dependent type.

## Reproduction

Export or transform one failed and one successful `path_payment_strict_send` operation through `TransformOperation()`. The failed path never rewrites `details["amount"]`, so it stays a quoted decimal string; the successful path replaces it with a numeric `float64`.

## Affected Code

- `internal/transform/operation.go:extractOperationDetails:660-697` — seeds `details["amount"]` as a string and rewrites it to `float64` only on success
- `internal/transform/operation.go:TransformOperation:29-100` — emits the mixed-typed details map without normalization
- `internal/transform/operation.go:transactionOperationWrapper.Details:1402-1416` — shows the alternate strict-send formatter keeps the field consistently string-typed, highlighting the live-path divergence

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPathPaymentStrictSendAmountTypeFlip`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `internal/transform/data_integrity_poc_test.go`
  3. Run: `go test ./internal/transform/... -run TestPathPaymentStrictSendAmountTypeFlip -v`
  4. Observe: failed rows serialize `details.amount` as `"0.0000000"` while successful rows serialize `details.amount` as `0.2`

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestPathPaymentStrictSendAmountTypeFlip(t *testing.T) {
	strictSendOp := xdr.Operation{
		SourceAccount: &testAccount3,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypePathPaymentStrictSend,
			PathPaymentStrictSendOp: &xdr.PathPaymentStrictSendOp{
				SendAsset:   nativeAsset,
				SendAmount:  1000000,
				Destination: testAccount4,
				DestAsset:   nativeAsset,
				DestMin:     500000,
			},
		},
	}

	failedOutput, err := TransformOperation(
		strictSendOp,
		0,
		makeStrictSendLedgerTransaction(strictSendOp, makeStrictSendResultPair(false)),
		1,
		makeLedgerCloseMeta(),
		"",
	)
	if err != nil {
		t.Fatalf("TransformOperation failed for failed strict-send tx: %v", err)
	}

	successOutput, err := TransformOperation(
		strictSendOp,
		0,
		makeStrictSendLedgerTransaction(strictSendOp, makeStrictSendResultPair(true)),
		1,
		makeLedgerCloseMeta(),
		"",
	)
	if err != nil {
		t.Fatalf("TransformOperation failed for successful strict-send tx: %v", err)
	}

	if _, ok := failedOutput.OperationDetails["amount"].(string); !ok {
		t.Fatalf("failed strict-send amount should stay a Go string before JSON encoding, got %T", failedOutput.OperationDetails["amount"])
	}
	if _, ok := successOutput.OperationDetails["amount"].(float64); !ok {
		t.Fatalf("successful strict-send amount should be a Go float64 before JSON encoding, got %T", successOutput.OperationDetails["amount"])
	}

	failedAmountJSON := marshalAmountField(t, failedOutput.OperationDetails)
	successAmountJSON := marshalAmountField(t, successOutput.OperationDetails)

	if string(failedAmountJSON) != `"0.0000000"` {
		t.Fatalf("failed strict-send JSON amount = %s, want %q", string(failedAmountJSON), "0.0000000")
	}
	if string(successAmountJSON) != `0.2` {
		t.Fatalf("successful strict-send JSON amount = %s, want 0.2", string(successAmountJSON))
	}
}

func makeStrictSendLedgerTransaction(operation xdr.Operation, result xdr.TransactionResultPair) ingest.LedgerTransaction {
	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: testAccount3,
			Memo:          xdr.Memo{},
			Operations:    []xdr.Operation{operation},
			Ext: xdr.TransactionExt{
				V: 0,
			},
		},
	}

	return ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result: result,
		UnsafeMeta: xdr.TransactionMeta{
			V:  1,
			V1: genericTxMeta,
		},
	}
}

func makeStrictSendResultPair(success bool) xdr.TransactionResultPair {
	result := xdr.PathPaymentStrictSendResult{
		Code: xdr.PathPaymentStrictSendResultCodePathPaymentStrictSendNoDestination,
	}
	resultCode := xdr.TransactionResultCodeTxFailed

	if success {
		result = xdr.PathPaymentStrictSendResult{
			Code: xdr.PathPaymentStrictSendResultCodePathPaymentStrictSendSuccess,
			Success: &xdr.PathPaymentStrictSendResultSuccess{
				Last: xdr.SimplePaymentResult{Amount: 2000000},
			},
		}
		resultCode = xdr.TransactionResultCodeTxSuccess
	}

	operationResults := []xdr.OperationResult{
		{
			Code: xdr.OperationResultCodeOpInner,
			Tr: &xdr.OperationResultTr{
				Type:                        xdr.OperationTypePathPaymentStrictSend,
				PathPaymentStrictSendResult: &result,
			},
		},
	}

	return xdr.TransactionResultPair{
		Result: xdr.TransactionResult{
			Result: xdr.TransactionResultResult{
				Code:    resultCode,
				Results: &operationResults,
			},
		},
	}
}

func marshalAmountField(t *testing.T, details map[string]interface{}) json.RawMessage {
	t.Helper()

	payload, err := json.Marshal(details)
	if err != nil {
		t.Fatalf("marshal details: %v", err)
	}

	var decoded map[string]json.RawMessage
	if err := json.Unmarshal(payload, &decoded); err != nil {
		t.Fatalf("unmarshal details: %v", err)
	}

	return decoded["amount"]
}
```

## Expected vs Actual Behavior

- **Expected**: `details.amount` should keep one JSON type for all `path_payment_strict_send` rows, even when the delivered amount is zero on failure.
- **Actual**: failed strict-send rows serialize `details.amount` as a JSON string, while successful rows serialize it as a JSON number.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `TransformOperation()` path for both failed and successful strict-send operations and inspects the serialized `details.amount` field.
2. Realistic preconditions: YES — failed `path_payment_strict_send` operations are normal on-chain outcomes, and the test uses the same transform entrypoint used during export.
3. Bug vs by-design: BUG — the type change is caused only by branch-local assignment types, and the alternate strict-send formatter in the same file keeps the field type stable.
4. Final severity: High — this is silent schema drift in exported operation data that can break downstream parsing and aggregations, but it does not change the arithmetic value itself.
5. In scope: YES — the exporter emits plausible but wrong-shaped output without crashing or logging an error.
6. Test correctness: CORRECT — the test does not assert a value it injected directly; it verifies the production transform result and the JSON representation derived from it.
7. Alternative explanations: NONE
8. Novelty: NOT ASSESSED HERE — duplicate handling is owned by the orchestrator rather than the final reviewer.

## Suggested Fix

Normalize strict-send `details.amount` to a single type in both branches. The least disruptive fix is to keep the current numeric contract and set the failure-path default with `utils.ConvertStroopValueToReal(0)`; if the intended contract is string-based, then both success and failure paths should consistently use `amount.String(...)` instead, and the strict-receive `source_amount` branch should be aligned the same way.
