# 002: contract event Parquet conversion drops populated `operation_id`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformContractEvent()` correctly computes a non-zero `operation_id` for operation-scoped Soroban contract events, but `ContractEventOutput.ToParquet()` never copies that field into the Parquet row. As a result, every exported contract-event Parquet row gets `operation_id = 0`, which silently breaks joins to operation data and collapses operation-scoped events into the same value used by transaction-level events.

## Root Cause

The JSON transform stores `OperationID` as `null.Int` on `ContractEventOutput`, and the Parquet schema declares an `operation_id` column on `ContractEventOutputParquet`. But the struct literal in `internal/transform/parquet_converter.go` omits `OperationID` entirely, so Go leaves the Parquet field at its zero value.

## Reproduction

During normal export, `cmd/export_contract_events.go` calls `TransformContractEvent()` for each transaction and appends each returned `ContractEventOutput` into the Parquet batch written by `WriteParquet()`. When an operation-scoped contract event is present, the JSON object contains a real TOID such as `131090201534537729`, but `ToParquet()` serializes the same row with `operation_id = 0`.

## Affected Code

- `internal/transform/contract_events.go:42-55` — operation-scoped events get `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)`
- `internal/transform/schema.go:641-656` — JSON schema exposes `OperationID null.Int`
- `internal/transform/schema_parquet.go:383-398` — Parquet schema declares `OperationID int64`
- `internal/transform/parquet_converter.go:425-441` — `ToParquet()` omits `OperationID`
- `cmd/export_contract_events.go:33-65` — export path writes transformed contract events through the Parquet converter

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractEventParquetDropsOperationID`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestContractEventParquetDropsOperationID -v`
  4. Observe that the operation-scoped event's Parquet `operation_id` is `0` instead of the non-zero JSON value.

### Test Body

```go
package transform

import "testing"

func TestContractEventParquetDropsOperationID(t *testing.T) {
	transactions, ledgerHeaders, err := makeContractEventTestInput()
	if err != nil {
		t.Fatalf("makeContractEventTestInput() error = %v", err)
	}

	outputs, err := TransformContractEvent(transactions[1], ledgerHeaders[1])
	if err != nil {
		t.Fatalf("TransformContractEvent() error = %v", err)
	}

	var operationEvent *ContractEventOutput
	for i := range outputs {
		if outputs[i].OperationID.Valid {
			operationEvent = &outputs[i]
			break
		}
	}

	if operationEvent == nil {
		t.Fatal("expected an operation-scoped contract event with a populated operation_id")
	}
	if operationEvent.OperationID.Int64 == 0 {
		t.Fatal("expected a non-zero operation_id for the operation-scoped contract event")
	}

	parquetOutput, ok := operationEvent.ToParquet().(ContractEventOutputParquet)
	if !ok {
		t.Fatal("ToParquet() did not return ContractEventOutputParquet")
	}

	if parquetOutput.OperationID != operationEvent.OperationID.Int64 {
		t.Fatalf("operation_id corrupted during parquet conversion: got %d, want %d", parquetOutput.OperationID, operationEvent.OperationID.Int64)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Operation-scoped contract-event rows should preserve the same non-zero `operation_id` in Parquet that the JSON transform computed.
- **Actual**: `ContractEventOutput.ToParquet()` leaves `operation_id` unset, so Parquet rows always serialize `0`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC starts from `TransformContractEvent()` using the repository's contract-event fixture and fails only when `ToParquet()` drops the populated `operation_id`.
2. Realistic preconditions: YES — any exported Soroban transaction with `OperationEvents` hits this exact path in `export_contract_events`.
3. Bug vs by-design: BUG — sibling converters copy their `OperationID` fields, the Parquet schema explicitly declares `operation_id`, and there is no code or documentation indicating that operation-scoped contract events should be zeroed.
4. Final severity: High — this is structural export corruption that breaks downstream joins and event classification, but it does not directly alter monetary values.
5. In scope: YES — production export logic silently emits wrong Parquet data.
6. Test correctness: CORRECT — the test uses production fixtures and code, asserts a real non-zero source value first, and then checks the converted row for the same value.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Populate the missing field in `ContractEventOutput.ToParquet()`, e.g. `OperationID: ceo.OperationID.Int64`, so operation-scoped rows preserve their computed TOIDs in Parquet output.
