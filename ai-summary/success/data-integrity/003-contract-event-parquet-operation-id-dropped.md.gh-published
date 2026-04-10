# 003: Contract event Parquet conversion drops populated `operation_id`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformContractEvent()` assigns a real TOID-backed `operation_id` to operation-scoped contract events, and the Parquet schema exposes an `operation_id` column for those rows. But `ContractEventOutput.ToParquet()` never copies that field, so every populated value is exported as `0` in Parquet.

This is silent structural corruption in the normal export path: `export_contract_events` collects transformed contract events and writes Parquet rows via `record.ToParquet()`, which erases the linkage from an operation-scoped event back to its originating operation.

## Root Cause

`ContractEventOutput` stores `OperationID` as `null.Int`, and `ContractEventOutputParquet` declares the target `OperationID int64` column. The converter in `internal/transform/parquet_converter.go` copies every other contract-event field but omits `OperationID`, so Go leaves the Parquet field at its zero value.

## Reproduction

During a normal `export_contract_events` run with Parquet enabled, any transaction that emits contract events inside `transactionEvents.OperationEvents` will hit this path. `TransformContractEvent()` sets a non-null operation TOID for those events, but `WriteParquet()` serializes `ContractEventOutputParquet` rows produced by `ToParquet()`, and the missing assignment turns each populated `operation_id` into `0`.

## Affected Code

- `internal/transform/contract_events.go:TransformContractEvent:42-54` — assigns `OperationID = null.IntFrom(operationID)` for operation-scoped contract events
- `internal/transform/schema.go:ContractEventOutput:641-656` — JSON/output schema stores `operation_id` as `null.Int`
- `internal/transform/schema_parquet.go:ContractEventOutputParquet:383-398` — Parquet schema declares the `operation_id` column
- `internal/transform/parquet_converter.go:ToParquet:425-441` — omits `OperationID` when building `ContractEventOutputParquet`
- `cmd/export_contract_events.go:Run:33-65` — normal export path accumulates contract events and writes the Parquet file
- `cmd/command_utils.go:WriteParquet:162-179` — writes `record.ToParquet()` output directly to the Parquet writer

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractEventToParquetDropsOperationID`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go test ./internal/transform/... -run TestContractEventToParquetDropsOperationID -v`.

### Test Body

```go
package transform

import (
	"testing"
)

func TestContractEventToParquetDropsOperationID(t *testing.T) {
	transactions, headers, err := makeContractEventTestInput()
	if err != nil {
		t.Fatalf("makeContractEventTestInput() error = %v", err)
	}

	events, err := TransformContractEvent(transactions[1], headers[1])
	if err != nil {
		t.Fatalf("TransformContractEvent() error = %v", err)
	}

	var operationEvent *ContractEventOutput
	for i := range events {
		if events[i].OperationID.Valid {
			operationEvent = &events[i]
			break
		}
	}
	if operationEvent == nil {
		t.Fatal("expected at least one operation-scoped contract event")
	}

	parquetRaw := operationEvent.ToParquet()
	parquet, ok := parquetRaw.(ContractEventOutputParquet)
	if !ok {
		t.Fatalf("ToParquet() returned unexpected type %T", parquetRaw)
	}

	if parquet.OperationID != operationEvent.OperationID.Int64 {
		t.Errorf("OperationID corrupted in Parquet conversion: got %d, want %d",
			parquet.OperationID, operationEvent.OperationID.Int64)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Operation-scoped contract events should preserve their populated TOID in the Parquet `operation_id` column.
- **Actual**: The Parquet row exports `operation_id = 0`, which is indistinguishable from an unset value and loses the event-to-operation linkage.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses the real `TransformContractEvent()` path on existing XDR fixtures, selects an event with a populated `OperationID`, and then calls the production `ToParquet()` converter.
2. Realistic preconditions: YES — operation-scoped contract events are normal Soroban output, and the existing transform tests already model them.
3. Bug vs by-design: BUG — the Parquet schema explicitly includes `operation_id`, and the JSON transform intentionally populates it for operation events; only the converter drops it.
4. Final severity: High — this is silent structural corruption in exported analytics data, not a direct monetary-field error.
5. In scope: YES — the exporter emits plausible but wrong Parquet rows without any error.
6. Test correctness: CORRECT — the test uses production fixtures and code paths, verifies the source field is populated first, and fails only because the converter zeroes the Parquet field.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Copy the populated field in `ContractEventOutput.ToParquet()`:

```go
OperationID: ceo.OperationID.Int64,
```

and add a converter test that covers both populated and unset `operation_id` cases so future copy-paste edits cannot silently regress this mapping.
