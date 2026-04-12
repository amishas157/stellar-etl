# 020: Failed Path-Payment Amount Fields Flip JSON Types

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports `history_operations.details.source_amount` and `history_operations.details.amount` with different JSON types depending on whether a path payment succeeds. Failed `path_payment_strict_receive` rows emit `"source_amount":"0.0000000"` and failed `path_payment_strict_send` rows emit `"amount":"0.0000000"`, while successful rows emit bare JSON numbers for the same keys.

This is a real export-structure bug, not just a downstream parsing quirk: the ETL itself serializes the same operation-detail key as a string in one branch and a number in another. That makes these fields unstable for typed consumers and diverges from the repo's own alternate operation-details formatter and Horizon's path-payment schema, both of which keep the fields as strings consistently.

## Root Cause

`extractOperationDetails()` seeds failed-path-payment sentinel values with `amount.String(0)` but overwrites them with `utils.ConvertStroopValueToReal(...)` only on success. Because `TransformOperation()` forwards the raw `map[string]interface{}` into both JSON-facing fields and the Parquet JSON-string converter without normalization, the Go `string`/`float64` split becomes a persisted export-type flip.

## Reproduction

Any normal export that includes failed and successful `path_payment_strict_receive` or `path_payment_strict_send` operations will hit this path. The failed rows serialize the unknown realized amount as a quoted decimal string, while successful rows serialize the same key as a bare JSON number, so batches containing both outcomes do not have a stable schema for those keys.

## Affected Code

- `internal/transform/operation.go:extractOperationDetails:619-699` — seeds failed path-payment amount fields with `amount.String(0)` and overwrites them with `float64` values only on success
- `internal/transform/operation.go:TransformOperation:29-100` — exports the mixed-typed details map unchanged via `OperationDetails` and `OperationDetailsJSON`
- `internal/transform/parquet_converter.go:OperationOutput.ToParquet:147-160` — preserves the same mixed JSON typing when serializing `details` for Parquet

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPathPaymentStrictReceiveSourceAmountTypeFlip`, `TestPathPaymentStrictSendAmountTypeFlip`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./... && go test ./internal/transform/... -run 'TestPathPaymentStrictReceiveSourceAmountTypeFlip|TestPathPaymentStrictSendAmountTypeFlip' -v` after creating the target test file with the body below.

### Test Body

```go
package transform

import (
	"encoding/json"
	"reflect"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func TestPathPaymentStrictReceiveSourceAmountTypeFlip(t *testing.T) {
	op := xdr.Operation{
		SourceAccount: &testAccount3,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypePathPaymentStrictReceive,
			PathPaymentStrictReceiveOp: &xdr.PathPaymentStrictReceiveOp{
				SendAsset:   nativeAsset,
				SendMax:     8951495900,
				Destination: testAccount4,
				DestAsset:   nativeAsset,
				DestAmount:  8951495900,
				Path:        []xdr.Asset{},
			},
		},
	}

	failedTx := buildPathPaymentLedgerTransaction(
		op,
		xdr.TransactionResultCodeTxFailed,
		xdr.OperationResultTr{
			Type: xdr.OperationTypePathPaymentStrictReceive,
			PathPaymentStrictReceiveResult: &xdr.PathPaymentStrictReceiveResult{
				Code: xdr.PathPaymentStrictReceiveResultCodePathPaymentStrictReceiveNoIssuer,
			},
		},
	)
	successTx := buildPathPaymentLedgerTransaction(
		op,
		xdr.TransactionResultCodeTxSuccess,
		xdr.OperationResultTr{
			Type: xdr.OperationTypePathPaymentStrictReceive,
			PathPaymentStrictReceiveResult: &xdr.PathPaymentStrictReceiveResult{
				Code: xdr.PathPaymentStrictReceiveResultCodePathPaymentStrictReceiveSuccess,
				Success: &xdr.PathPaymentStrictReceiveResultSuccess{
					Last: xdr.SimplePaymentResult{Amount: 8946764349},
				},
			},
		},
	)

	assertOperationDetailTypeStable(t, op, "source_amount", "path_payment_strict_receive", failedTx, successTx)
}

func TestPathPaymentStrictSendAmountTypeFlip(t *testing.T) {
	op := xdr.Operation{
		SourceAccount: &testAccount3,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypePathPaymentStrictSend,
			PathPaymentStrictSendOp: &xdr.PathPaymentStrictSendOp{
				SendAsset:   nativeAsset,
				SendAmount:  1598182,
				Destination: testAccount4,
				DestAsset:   nativeAsset,
				DestMin:     4280460538,
				Path:        []xdr.Asset{},
			},
		},
	}

	failedTx := buildPathPaymentLedgerTransaction(
		op,
		xdr.TransactionResultCodeTxFailed,
		xdr.OperationResultTr{
			Type: xdr.OperationTypePathPaymentStrictSend,
			PathPaymentStrictSendResult: &xdr.PathPaymentStrictSendResult{
				Code: xdr.PathPaymentStrictSendResultCodePathPaymentStrictSendNoIssuer,
			},
		},
	)
	successTx := buildPathPaymentLedgerTransaction(
		op,
		xdr.TransactionResultCodeTxSuccess,
		xdr.OperationResultTr{
			Type: xdr.OperationTypePathPaymentStrictSend,
			PathPaymentStrictSendResult: &xdr.PathPaymentStrictSendResult{
				Code: xdr.PathPaymentStrictSendResultCodePathPaymentStrictSendSuccess,
				Success: &xdr.PathPaymentStrictSendResultSuccess{
					Last: xdr.SimplePaymentResult{Amount: 5000000000},
				},
			},
		},
	)

	assertOperationDetailTypeStable(t, op, "amount", "path_payment_strict_send", failedTx, successTx)
}

func assertOperationDetailTypeStable(
	t *testing.T,
	op xdr.Operation,
	detailKey string,
	opType string,
	failedTx ingest.LedgerTransaction,
	successTx ingest.LedgerTransaction,
) {
	t.Helper()

	ledgerCloseMeta := makeLedgerCloseMeta()

	failedOutput, err := TransformOperation(op, 0, failedTx, 1, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation failed transaction: %v", err)
	}
	successOutput, err := TransformOperation(op, 0, successTx, 1, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation successful transaction: %v", err)
	}

	failedValue := failedOutput.OperationDetailsJSON[detailKey]
	successValue := successOutput.OperationDetailsJSON[detailKey]
	failedJSON, err := json.Marshal(failedOutput.OperationDetailsJSON)
	if err != nil {
		t.Fatalf("marshal failed output: %v", err)
	}
	successJSON, err := json.Marshal(successOutput.OperationDetailsJSON)
	if err != nil {
		t.Fatalf("marshal successful output: %v", err)
	}

	if reflect.TypeOf(failedValue) != reflect.TypeOf(successValue) {
		t.Fatalf(
			"%s details[%q] changed type across transaction outcomes: failed=%T success=%T; failed_json=%s success_json=%s",
			opType,
			detailKey,
			failedValue,
			successValue,
			failedJSON,
			successJSON,
		)
	}
}

func buildPathPaymentLedgerTransaction(
	op xdr.Operation,
	txCode xdr.TransactionResultCode,
	opResult xdr.OperationResultTr,
) ingest.LedgerTransaction {
	operationResults := []xdr.OperationResult{
		{
			Code: xdr.OperationResultCodeOpInner,
			Tr:   &opResult,
		},
	}

	txMeta := utils.CreateSampleTxMeta(29, lpAssetA, lpAssetB)

	return ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount3,
					Memo:          xdr.Memo{},
					Operations:    []xdr.Operation{op},
					Ext:           xdr.TransactionExt{V: 0},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			Result: xdr.TransactionResult{
				Result: xdr.TransactionResultResult{
					Code:    txCode,
					Results: &operationResults,
				},
			},
		},
		UnsafeMeta: xdr.TransactionMeta{
			V:  1,
			V1: txMeta,
		},
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Each path-payment detail key keeps one stable JSON type across failed and successful rows, regardless of whether the exporter chooses strings or numbers.
- **Actual**: Failed rows serialize the sentinel amount as a JSON string while successful rows serialize the same key as a JSON number.

## Adversarial Review

1. Exercises claimed bug: YES — the test calls the production `TransformOperation()` path and inspects the exported JSON-facing details payload after marshalling.
2. Realistic preconditions: YES — failed and successful path payments are standard Stellar outcomes, and the test uses real XDR operation/result types rather than mocks or private hooks.
3. Bug vs by-design: BUG — the repo's alternate `transactionOperationWrapper.Details()` path and Horizon's operation schema keep these amount fields as strings consistently, so the mixed `string`/`float64` export here is not an intentional contract.
4. Final severity: High — this corrupts the structure of exported operation-details data for normal path-payment rows and makes the same key unstable for typed downstream consumers.
5. In scope: YES — the ETL emits wrong persisted output without crashing or reporting an error.
6. Test correctness: CORRECT — the test drives the real transform code, compares the same key across realistic success/failure outcomes, and includes the final serialized JSON in the failure message to show the actual export difference.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Normalize path-payment detail amounts to a single representation on both branches. The least risky fix is to mirror Horizon and the repo's `transactionOperationWrapper.Details()` implementation by using `amount.String(...)` for these path-payment amount fields in both failed and successful cases, then add regression coverage for failed `path_payment_strict_receive` and `path_payment_strict_send`.
