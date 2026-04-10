# 001: Contract event Parquet operation ID dropped

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

Operation-scoped contract events carry a real TOID-backed `operation_id` in the JSON export, but the Parquet conversion drops that populated value and writes `0` instead. This silently corrupts the join key downstream systems need to relate contract events back to their parent operations.

## Root Cause

`TransformContractEvent()` correctly sets `ContractEventOutput.OperationID` for operation events, but `ContractEventOutput.ToParquet()` never copies that field into `ContractEventOutputParquet`. Because the Parquet struct uses a plain `int64`, the omitted assignment falls back to Go's zero value and every exported row gets `operation_id = 0`.

## Reproduction

Any normal `export_contract_events --write-parquet` run that includes Soroban operation events will hit this path. The transform layer produces a non-zero `operation_id`, then the Parquet conversion erases it before `cmd/export_contract_events.go` writes the record.

## Affected Code

- `internal/transform/contract_events.go:TransformContractEvent:42-52` - computes and assigns non-zero operation IDs for operation-scoped events
- `internal/transform/parquet_converter.go:ContractEventOutput.ToParquet:425-441` - omits `OperationID` when building the Parquet row
- `cmd/export_contract_events.go:Run:50-65` - writes the corrupted Parquet rows when `--write-parquet` is enabled

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractEventToParquetZeroesOperationID`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import "testing"

func TestContractEventToParquetZeroesOperationID(t *testing.T) {
	transactions, ledgerHeaders, err := makeContractEventTestInput()
	if err != nil {
		t.Fatalf("makeContractEventTestInput() error = %v", err)
	}

	for i := range transactions {
		events, err := TransformContractEvent(transactions[i], ledgerHeaders[i])
		if err != nil {
			t.Fatalf("TransformContractEvent() error = %v", err)
		}

		for _, event := range events {
			if !event.OperationID.Valid || event.OperationID.Int64 == 0 {
				continue
			}

			expectedOperationID := event.OperationID.Int64
			parquetEvent, ok := event.ToParquet().(ContractEventOutputParquet)
			if !ok {
				t.Fatalf("ToParquet() returned unexpected type %T", event.ToParquet())
			}

			if parquetEvent.OperationID == 0 {
				t.Logf("BUG CONFIRMED: Parquet OperationID is 0, expected %d", expectedOperationID)
				return
			}

			t.Fatalf("OperationID was preserved as %d; hypothesis not demonstrated", parquetEvent.OperationID)
		}
	}

	t.Fatal("fixture did not produce any operation-scoped contract event with a non-zero OperationID")
}
```

## Expected vs Actual Behavior

- **Expected**: Operation-scoped Parquet rows preserve the same non-zero `operation_id` that `TransformContractEvent()` computed.
- **Actual**: Parquet rows serialize `operation_id` as `0`, making populated operation IDs indistinguishable from absent ones.

## Adversarial Review

1. Exercises claimed bug: YES - the PoC runs the real `TransformContractEvent()` fixture path, then converts a genuinely populated operation-scoped event with `ToParquet()`.
2. Realistic preconditions: YES - Soroban operation events are standard output for normal contract execution, and `export_contract_events --write-parquet` always passes through this converter.
3. Bug vs by-design: BUG - although Parquet nullable integers elsewhere degrade missing values to `0`, this path drops a populated ID that the producer already computed.
4. Final severity: High - this is structural data corruption of a join key, not direct monetary corruption.
5. In scope: YES - it is a concrete production code path producing wrong exported data.
6. Test correctness: CORRECT - the test uses production fixtures and checks a non-zero source value against the emitted Parquet value.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Populate `OperationID` in `ContractEventOutput.ToParquet()`, using the underlying `null.Int` value so operation-scoped rows preserve their TOID while nil rows continue following the existing null-handling convention.
