# 001: Contract Event Parquet Drops Operation TOID

**Date**: 2026-04-11
**Severity**: High
**Impact**: structural data corruption
**Subsystem**: toid
**Final review by**: gpt-5.4, high

## Summary

Operation-scoped contract events compute a non-zero TOID in `TransformContractEvent`, and the JSON export retains it in `operation_id`. The Parquet conversion path silently drops that populated value, so every affected Parquet row emits `operation_id = 0` instead of the real operation TOID.

## Root Cause

`ContractEventOutput.ToParquet()` constructs a `ContractEventOutputParquet` literal but never assigns `OperationID`. Because the Parquet schema models `operation_id` as an `int64`, Go writes the field's zero value even when `ContractEventOutput.OperationID` was populated by the transform step.

## Reproduction

Any ledger containing an operation-scoped Soroban contract event will hit this path when `export_contract_events --write-parquet` is used. The transform path computes the operation TOID from `(ledger sequence, transaction index, operation index)`, but the export path loses it during `ToParquet()`.

## Affected Code

- `internal/transform/contract_events.go:21-67` — operation-scoped events compute and populate `OperationID`
- `internal/transform/schema.go:640-657` — JSON schema carries `operation_id` as `null.Int`
- `internal/transform/schema_parquet.go:382-399` — Parquet schema exposes a physical `operation_id` column
- `internal/transform/parquet_converter.go:425-441` — contract-event Parquet conversion omits `OperationID`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractEventParquetDropsOperationTOID`
- **Test language**: `go`
- **How to run**: Create the target test file with the test body below, then run `go test ./internal/transform/... -run TestContractEventParquetDropsOperationTOID -v`.

### Test Body

```go
package transform

import "testing"

func TestContractEventParquetDropsOperationTOID(t *testing.T) {
	transactions, headers, err := makeContractEventTestInput()
	if err != nil {
		t.Fatalf("makeContractEventTestInput(): %v", err)
	}

	foundOperationScopedEvent := false

	for i := range transactions {
		events, err := TransformContractEvent(transactions[i], headers[i])
		if err != nil {
			t.Fatalf("TransformContractEvent(): %v", err)
		}

		for _, event := range events {
			if !event.OperationID.Valid {
				continue
			}

			foundOperationScopedEvent = true

			parquetRow := event.ToParquet().(ContractEventOutputParquet)
			if parquetRow.OperationID != 0 {
				t.Fatalf("expected buggy parquet operation_id to be 0, got %d", parquetRow.OperationID)
			}
			if parquetRow.OperationID == event.OperationID.Int64 {
				t.Fatalf("expected parquet operation_id to differ from populated JSON operation_id %d", event.OperationID.Int64)
			}

			t.Logf(
				"operation-scoped event lost operation_id: json=%d parquet=%d",
				event.OperationID.Int64,
				parquetRow.OperationID,
			)
		}
	}

	if !foundOperationScopedEvent {
		t.Fatal("expected at least one operation-scoped contract event with a populated operation_id")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Parquet `operation_id` should match the non-zero TOID already computed on the JSON row for every operation-scoped contract event.
- **Actual**: Parquet `operation_id` is always `0` because the converter omits the field assignment entirely.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses the existing contract-event fixtures, calls `TransformContractEvent`, then sends real operation-scoped rows through `ToParquet()`.
2. Realistic preconditions: YES — any real operation-scoped Soroban contract event exported with `--write-parquet` uses this exact path.
3. Bug vs by-design: BUG — the schema declares `operation_id`, the transform populates it, and sibling Parquet converters copy comparable IDs; only this converter drops it.
4. Final severity: High — this corrupts a structural join key, breaking linkage between contract events and parent operations without any export failure.
5. In scope: YES — this is silent exported data corruption in the ETL pipeline.
6. Test correctness: CORRECT — it asserts a populated production `OperationID` before conversion and demonstrates that the Parquet converter alone zeroes it out.
7. Alternative explanations: NONE — the observed zero comes directly from the omitted field assignment, not from upstream transform logic.
8. Novelty: NOT ASSESSED — duplicate handling is performed by the orchestrator.

## Suggested Fix

Add `OperationID: ceo.OperationID.Int64,` to the `ContractEventOutputParquet` struct literal in `ContractEventOutput.ToParquet()`.
