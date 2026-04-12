# H001: Failed `path_payment_strict_receive` rows flip `details.source_amount` from number to string

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details.source_amount` should keep a stable JSON type for all `path_payment_strict_receive` rows. If a failed path payment has no realized send amount, the exporter should emit either a numeric `0` or `null`, not a quoted decimal string, so successful and failed rows can be consumed with the same schema.

## Mechanism

`extractOperationDetails()` initializes `details["source_amount"]` with `amount.String(0)`, which is a string, and only overwrites it with `utils.ConvertStroopValueToReal(...)` when the transaction succeeds. `TransformOperation()` then writes that mixed-typed map directly into both `OperationDetails` and `OperationDetailsJSON`, so failed rows serialize `"source_amount":"0.0000000"` while successful rows serialize `"source_amount":894.6764349`.

## Trigger

1. Export a ledger containing a failed `path_payment_strict_receive` operation.
2. Compare its `history_operations.details.source_amount` field with a successful `path_payment_strict_receive` row.
3. Observe that the failed row emits a quoted decimal string while the successful row emits a JSON number for the same key.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:619-656` — seeds `source_amount` with `amount.String(0)` and conditionally overwrites it only on success
- `internal/transform/operation.go:TransformOperation:29-57` — forwards the mixed-typed details map into exported JSON
- `internal/transform/operation_test.go:1099-1129` — asserts the successful branch exports numeric `source_amount`, but does not cover the failed branch

## Evidence

The live exporter's strict-receive branch uses two different Go types for the same key: a string sentinel on the default path and a `float64` on the success path. The successful-path test fixture confirms the repository already treats `source_amount` as a number in at least one branch of this operation family.

## Anti-Evidence

`OperationDetails` is a free-form `map[string]interface{}`, so the project may tolerate some per-row type variation. Reviewer may also decide that the upstream Horizon convention of defaulting failed path-payment outcomes to zero makes the quoted zero acceptable even though the successful branch is numeric.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `extractOperationDetails()` in `operation.go` from line 619 through the `PathPaymentStrictReceive` case. Confirmed that line 632 sets `details["source_amount"] = amount.String(0)` which returns Go type `string` (value `"0.0000000"`), while line 655 inside the `transaction.Result.Successful()` guard overwrites with `utils.ConvertStroopValueToReal(result.SendAmount())` which returns Go type `float64`. The `map[string]interface{}` is passed directly into `OperationOutput.OperationDetails` and `OperationDetailsJSON` at lines 92 and 97, so JSON serialization preserves the Go type distinction. The identical pattern exists for `details["amount"]` in `PathPaymentStrictSend` (lines 672 vs 696).

### Code Paths Examined

- `internal/transform/operation.go:extractOperationDetails:619-656` — Confirmed: line 632 assigns `amount.String(0)` (string `"0.0000000"`), line 655 assigns `utils.ConvertStroopValueToReal()` (float64). The guard at line 641 is `transaction.Result.Successful()`, so failed transactions retain the string sentinel.
- `internal/transform/operation.go:TransformOperation:30-100` — Confirmed: line 54 calls `extractOperationDetails()`, lines 92/97 assign the result map to both `OperationDetails` and `OperationDetailsJSON` without any type normalization.
- `github.com/stellar/go-stellar-sdk/amount/main.go:134-158` — Confirmed: `amount.String()` returns Go `string` via `big.Rat.FloatString(7)`.
- `internal/utils/main.go:ConvertStroopValueToReal` — Returns Go `float64` via division by 1e7.
- `internal/transform/operation.go:660-699` — Same pattern in `PathPaymentStrictSend`: `details["amount"] = amount.String(0)` (string) at line 672, overwritten to `utils.ConvertStroopValueToReal()` (float64) at line 696 on success.
- `internal/transform/operation.go:1364-1416` — Alternative `transactionOperationWrapper.Details()` method uses `amount.String()` consistently for ALL amounts (no type flip), confirming string-only is the canonical upstream pattern.

### Findings

1. **Primary bug confirmed**: `source_amount` in `path_payment_strict_receive` exports as JSON string `"0.0000000"` for failed transactions and JSON number (e.g., `894.6764349`) for successful transactions. Any downstream consumer (BigQuery, Parquet ingestion, schema-enforcing pipelines) that infers column type from a batch of rows will encounter a type conflict.

2. **Same bug in sibling operation**: `details["amount"]` in `path_payment_strict_send` has the identical string-to-float64 type flip (lines 672 vs 696). The PoC should test both.

3. **Additional type inconsistency**: `destination_min` in StrictSend (line 674) uses `amount.String()` (always string), while `source_max` in StrictReceive (line 633) uses `ConvertStroopValueToReal()` (always float64). These are different keys so no per-key type flip, but the inconsistency across the amount-field family is notable.

4. **Root cause**: `extractOperationDetails()` predominantly uses `ConvertStroopValueToReal()` (float64) for amount fields, but the sentinel/default values for path payment unknowns use `amount.String(0)` (string). The alternative `transactionOperationWrapper.Details()` method (used for effects, not exports) is consistent — it uses `amount.String()` for everything.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Construct a `path_payment_strict_receive` operation with a failed transaction result (e.g., `xdr.TransactionResultCodeTxFailed`). Use existing test helpers like `utils.CreateSampleTx()` or construct a minimal `ingest.LedgerTransaction` with `Result.Successful()` returning false.
- **Steps**:
  1. Call `TransformOperation()` with the failed `path_payment_strict_receive` operation.
  2. Extract `outputDetails["source_amount"]` from the result.
  3. Check its Go type using a type switch or `reflect.TypeOf()`.
  4. Also test the successful case and verify the type is `float64`.
- **Assertion**: Assert that the Go type of `details["source_amount"]` is the same (either both `string` or both `float64`) regardless of transaction success. Currently, failed returns `string` and successful returns `float64`. Optionally, repeat for `path_payment_strict_send` with `details["amount"]`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestPathPaymentStrictReceiveSourceAmountTypeFlip" and "TestPathPaymentStrictSendAmountTypeFlip"
**Test Language**: Go

### Demonstration

Both tests confirm the type-flip bug. For `path_payment_strict_receive`, `details["source_amount"]` is Go type `string` (value `"0.0000000"`) when the transaction fails, but Go type `float64` (value `894.6764349`) when it succeeds. The identical pattern holds for `path_payment_strict_send` where `details["amount"]` flips from `string` to `float64`. This means JSON serialization produces quoted strings vs bare numbers for the same key depending on transaction success, breaking schema-enforcing downstream consumers.

### Test Body

```go
package transform

import (
	"fmt"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestPathPaymentStrictReceiveSourceAmountTypeFlip demonstrates that
// details["source_amount"] is a string for failed transactions but a float64
// for successful ones — a JSON type inconsistency (H001).
func TestPathPaymentStrictReceiveSourceAmountTypeFlip(t *testing.T) {
	ledgerCloseMeta := makeLedgerCloseMeta()

	// Build the path_payment_strict_receive operation
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

	// --- Case 1: Failed transaction ---
	failedTx := buildTxWithPathPaymentResult(false, xdr.OperationTypePathPaymentStrictReceive, nil)
	failedOutput, err := TransformOperation(op, 0, failedTx, 1, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation (failed) returned error: %v", err)
	}
	failedSourceAmount := failedOutput.OperationDetails["source_amount"]
	failedType := fmt.Sprintf("%T", failedSourceAmount)

	// --- Case 2: Successful transaction ---
	successResult := &xdr.PathPaymentStrictReceiveResult{
		Code: xdr.PathPaymentStrictReceiveResultCodePathPaymentStrictReceiveSuccess,
		Success: &xdr.PathPaymentStrictReceiveResultSuccess{
			Last: xdr.SimplePaymentResult{Amount: 8946764349},
		},
	}
	successTx := buildTxWithPathPaymentResult(true, xdr.OperationTypePathPaymentStrictReceive, successResult)
	successOutput, err := TransformOperation(op, 0, successTx, 1, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation (success) returned error: %v", err)
	}
	successSourceAmount := successOutput.OperationDetails["source_amount"]
	successType := fmt.Sprintf("%T", successSourceAmount)

	t.Logf("Failed  tx source_amount: value=%v type=%s", failedSourceAmount, failedType)
	t.Logf("Success tx source_amount: value=%v type=%s", successSourceAmount, successType)

	// The bug: the types differ between failed and successful transactions
	if failedType != successType {
		t.Errorf("TYPE FLIP CONFIRMED: failed tx source_amount type is %s, successful tx type is %s — same key has inconsistent JSON types", failedType, successType)
	}
}

// TestPathPaymentStrictSendAmountTypeFlip demonstrates the same type-flip bug
// for details["amount"] in path_payment_strict_send operations (sibling bug).
func TestPathPaymentStrictSendAmountTypeFlip(t *testing.T) {
	ledgerCloseMeta := makeLedgerCloseMeta()

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

	// --- Case 1: Failed transaction ---
	failedTx := buildTxWithPathPaymentResult(false, xdr.OperationTypePathPaymentStrictSend, nil)
	failedOutput, err := TransformOperation(op, 0, failedTx, 1, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation (failed) returned error: %v", err)
	}
	failedAmount := failedOutput.OperationDetails["amount"]
	failedType := fmt.Sprintf("%T", failedAmount)

	// --- Case 2: Successful transaction ---
	successResult := &xdr.PathPaymentStrictSendResult{
		Code: xdr.PathPaymentStrictSendResultCodePathPaymentStrictSendSuccess,
		Success: &xdr.PathPaymentStrictSendResultSuccess{
			Last: xdr.SimplePaymentResult{Amount: 5000000000},
		},
	}
	successTx := buildTxWithPathPaymentResult(true, xdr.OperationTypePathPaymentStrictSend, successResult)
	successOutput, err := TransformOperation(op, 0, successTx, 1, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation (failed) returned error: %v", err)
	}
	successAmount := successOutput.OperationDetails["amount"]
	successType := fmt.Sprintf("%T", successAmount)

	t.Logf("Failed  tx amount: value=%v type=%s", failedAmount, failedType)
	t.Logf("Success tx amount: value=%v type=%s", successAmount, successType)

	if failedType != successType {
		t.Errorf("TYPE FLIP CONFIRMED: failed tx amount type is %s, successful tx type is %s — same key has inconsistent JSON types", failedType, successType)
	}
}

// buildTxWithPathPaymentResult constructs a minimal ingest.LedgerTransaction
// with a single operation result for a path payment operation.
func buildTxWithPathPaymentResult(successful bool, opType xdr.OperationType, successResult interface{}) ingest.LedgerTransaction {
	resultCode := xdr.TransactionResultCodeTxFailed
	if successful {
		resultCode = xdr.TransactionResultCodeTxSuccess
	}

	dummyOp := xdr.Operation{
		SourceAccount: &testAccount3,
		Body: xdr.OperationBody{
			Type:           xdr.OperationTypeBumpSequence,
			BumpSequenceOp: &xdr.BumpSequenceOp{},
		},
	}

	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: testAccount3,
			Memo:          xdr.Memo{},
			Operations:    []xdr.Operation{dummyOp},
			Ext: xdr.TransactionExt{
				V: 0,
				SorobanData: &xdr.SorobanTransactionData{
					Ext:       xdr.SorobanTransactionDataExt{V: 0},
					Resources: xdr.SorobanResources{Footprint: xdr.LedgerFootprint{}},
				},
			},
		},
	}

	var opResultTr xdr.OperationResultTr
	switch opType {
	case xdr.OperationTypePathPaymentStrictReceive:
		if successful {
			result := successResult.(*xdr.PathPaymentStrictReceiveResult)
			opResultTr = xdr.OperationResultTr{
				Type:                           xdr.OperationTypePathPaymentStrictReceive,
				PathPaymentStrictReceiveResult: result,
			}
		} else {
			opResultTr = xdr.OperationResultTr{
				Type: xdr.OperationTypePathPaymentStrictReceive,
				PathPaymentStrictReceiveResult: &xdr.PathPaymentStrictReceiveResult{
					Code: xdr.PathPaymentStrictReceiveResultCodePathPaymentStrictReceiveNoIssuer,
				},
			}
		}
	case xdr.OperationTypePathPaymentStrictSend:
		if successful {
			result := successResult.(*xdr.PathPaymentStrictSendResult)
			opResultTr = xdr.OperationResultTr{
				Type:                        xdr.OperationTypePathPaymentStrictSend,
				PathPaymentStrictSendResult: result,
			}
		} else {
			opResultTr = xdr.OperationResultTr{
				Type: xdr.OperationTypePathPaymentStrictSend,
				PathPaymentStrictSendResult: &xdr.PathPaymentStrictSendResult{
					Code: xdr.PathPaymentStrictSendResultCodePathPaymentStrictSendNoIssuer,
				},
			}
		}
	}

	operationResults := []xdr.OperationResult{
		{
			Code: xdr.OperationResultCodeOpInner,
			Tr:   &opResultTr,
		},
	}

	txMeta := utils.CreateSampleTxMeta(29, lpAssetA, lpAssetB)

	return ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result: xdr.TransactionResultPair{
			Result: xdr.TransactionResult{
				Result: xdr.TransactionResultResult{
					Code:    resultCode,
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

### Test Output

```
=== RUN   TestPathPaymentStrictReceiveSourceAmountTypeFlip
    data_integrity_poc_test.go:58: Failed  tx source_amount: value=0.0000000 type=string
    data_integrity_poc_test.go:59: Success tx source_amount: value=894.6764349 type=float64
    data_integrity_poc_test.go:63: TYPE FLIP CONFIRMED: failed tx source_amount type is string, successful tx type is float64 — same key has inconsistent JSON types
--- FAIL: TestPathPaymentStrictReceiveSourceAmountTypeFlip (0.00s)
=== RUN   TestPathPaymentStrictSendAmountTypeFlip
    data_integrity_poc_test.go:111: Failed  tx amount: value=0.0000000 type=string
    data_integrity_poc_test.go:112: Success tx amount: value=500 type=float64
    data_integrity_poc_test.go:115: TYPE FLIP CONFIRMED: failed tx amount type is string, successful tx type is float64 — same key has inconsistent JSON types
--- FAIL: TestPathPaymentStrictSendAmountTypeFlip (0.00s)
FAIL
```
