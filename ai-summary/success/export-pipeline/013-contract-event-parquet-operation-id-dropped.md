# 013: Contract event Parquet export drops operation IDs

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformContractEvent()` correctly computes non-zero `operation_id` values for operation-scoped Soroban events, and JSON export preserves them. The Parquet path silently rewrites those same rows to `operation_id=0` because `ContractEventOutput.ToParquet()` never copies the populated field into `ContractEventOutputParquet`.

## Root Cause

`internal/transform/parquet_converter.go` constructs `ContractEventOutputParquet` field-by-field but omits `OperationID`, so Go leaves the Parquet struct field at its zero value. `cmd/export_contract_events.go` passes transformed contract events into `WriteParquet()`, which serializes `record.ToParquet()` directly, so the omission reaches emitted Parquet files.

## Reproduction

On any ledger containing Soroban operation events, `TransformContractEvent()` sets `OperationID` with a TOID derived from ledger sequence, transaction index, and operation index. When those rows are converted for Parquet output, the exported `operation_id` column becomes `0` instead of the real TOID, breaking joins between contract events and operations while leaving the row otherwise plausible.

## Affected Code

- `internal/transform/contract_events.go:21-67` — assigns non-zero `OperationID` for operation-scoped events
- `internal/transform/parquet_converter.go:425-442` — omits `OperationID` when building `ContractEventOutputParquet`
- `internal/transform/schema_parquet.go:382-399` — declares an `operation_id` Parquet column that is never populated
- `cmd/command_utils.go:162-180` — writes `record.ToParquet()` into Parquet output
- `cmd/export_contract_events.go:32-65` — routes contract-event exports through the broken Parquet conversion path

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractEventParquetZeroesOperationID`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import "testing"

func TestContractEventParquetZeroesOperationID(t *testing.T) {
	hardCodedTransaction, hardCodedLedgerHeader, err := makeContractEventTestInput()
	if err != nil {
		t.Fatalf("failed to create test input: %v", err)
	}

	foundOperationEvent := false

	for txIdx, tx := range hardCodedTransaction {
		outputs, err := TransformContractEvent(tx, hardCodedLedgerHeader[txIdx])
		if err != nil {
			t.Fatalf("TransformContractEvent failed for tx %d: %v", txIdx, err)
		}

		for evtIdx, output := range outputs {
			parquetRaw := output.ToParquet()
			parquetOutput, ok := parquetRaw.(ContractEventOutputParquet)
			if !ok {
				t.Fatalf("ToParquet() returned unexpected type %T", parquetRaw)
			}

			if output.OperationID.Valid && output.OperationID.Int64 != 0 {
				foundOperationEvent = true

				if parquetOutput.OperationID != output.OperationID.Int64 {
					t.Errorf(
						"tx[%d] event[%d]: parquet OperationID=%d, want %d",
						txIdx,
						evtIdx,
						parquetOutput.OperationID,
						output.OperationID.Int64,
					)
				}
			}
		}
	}

	if !foundOperationEvent {
		t.Fatal("test setup error: no operation-scoped events with non-zero OperationID found in fixtures")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Operation-scoped contract events preserve their non-zero JSON `operation_id` TOIDs in Parquet output.
- **Actual**: Parquet conversion emits `operation_id=0` for those rows because the converter never copies the populated field.

## Adversarial Review

1. Exercises claimed bug: YES — the test uses the real `TransformContractEvent()` fixture path, then the same `ToParquet()` conversion that `WriteParquet()` serializes in production.
2. Realistic preconditions: YES — any Soroban ledger with operation-scoped events hits this path during normal `export_contract_events --write-parquet` usage.
3. Bug vs by-design: BUG — the schema defines `operation_id`, JSON output already carries it, and only the converter omits the assignment.
4. Final severity: High — this is structural data corruption in a join key, not a direct monetary miscalculation.
5. In scope: YES — it is a concrete production code path producing silently wrong exported data.
6. Test correctness: CORRECT — it asserts equality between the real transformed JSON field and the Parquet struct written by the production converter, and it verifies the fixture actually contains non-zero operation-scoped events.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Populate `OperationID` in `ContractEventOutput.ToParquet()` (for example, `OperationID: ceo.OperationID.Int64`) so operation-scoped rows preserve their computed TOIDs in Parquet output. A follow-up hardening pass should also decide how absent `operation_id` values ought to be represented in Parquet.
