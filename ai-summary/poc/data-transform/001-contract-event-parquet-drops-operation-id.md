# H001: Contract-event Parquet rows drop operation IDs

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Operation-scoped contract events should preserve the same `operation_id` in Parquet that the JSON transform already computes. Transaction-level and diagnostic events with no operation should remain null/absent, not collapse into the same value as real operation-scoped events.

## Mechanism

`TransformContractEvent()` computes and stores `OperationID` for operation events, and the JSON schema keeps that field nullable. But `ContractEventOutput.ToParquet()` never assigns `OperationID` even though `ContractEventOutputParquet` declares the column, so every Parquet row gets the zero value `0`. That silently breaks joins from `history_contract_events` to `history_operations` and also erases the distinction between transaction-level events and operation-level events.

## Trigger

Export any Soroban transaction whose `TransactionMetaV4.OperationEvents` contains a contract event. The JSON row will carry a non-null operation TOID, but the Parquet row for the same event will contain `operation_id = 0`.

## Target Code

- `internal/transform/contract_events.go:42-55` — operation events are assigned `null.IntFrom(operationID)`
- `internal/transform/schema.go:641-656` — JSON output keeps `OperationID` as `null.Int`
- `internal/transform/schema_parquet.go:383-398` — Parquet schema declares a concrete `operation_id` column
- `internal/transform/parquet_converter.go:425-441` — converter omits `OperationID` entirely
- `internal/transform/contract_events_test.go:111-127` — existing transform tests already expect a non-null operation ID for operation-scoped events

## Evidence

`TransformContractEvent()` sets `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)` for operation events at `contract_events.go:50-52`. The test fixture in `contract_events_test.go:111-127` expects `OperationID: null.IntFrom(131090201534537729)` for one such event. But `ContractEventOutput.ToParquet()` copies every other field and never writes `OperationID`, even though `ContractEventOutputParquet` has `OperationID int64`.

## Anti-Evidence

Some contract-event rows legitimately have no operation context, so a nullable representation is required somewhere in the pipeline. But that makes the current omission worse, not safer: Parquet now conflates "no operation" with "operation-scoped event whose ID was never serialized."

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `TransformContractEvent()` through `parseDiagnosticEvent()` to `ContractEventOutput.ToParquet()`. Confirmed that operation-scoped events correctly receive a non-null `OperationID` (via `null.IntFrom(operationID)` at `contract_events.go:52`), the JSON schema declares the field as `null.Int` (`schema.go:656`), and the Parquet schema declares a corresponding `int64` column (`schema_parquet.go:398`). However, the `ToParquet()` converter at `parquet_converter.go:425-441` constructs the Parquet struct with all 14 other fields but omits `OperationID` entirely, causing it to default to Go's zero value (`0`) for every row. This is a textbook Pattern 2 (Parquet field mapping) copy-paste omission.

### Code Paths Examined

- `internal/transform/contract_events.go:42-55` — `TransformContractEvent()` loops over `transactionEvents.OperationEvents`, computes `operationID` via `toid.New()`, and sets `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)`. Confirmed this correctly populates the JSON struct field.
- `internal/transform/contract_events.go:167-242` — `parseDiagnosticEvent()` constructs the `ContractEventOutput` struct but does NOT set `OperationID` (it's set by the caller for operation-scoped events only). Transaction-level and diagnostic events leave it as the zero-value `null.Int{}`.
- `internal/transform/schema.go:641-657` — `ContractEventOutput` declares `OperationID null.Int` as the last field. JSON serialization correctly handles null vs non-null.
- `internal/transform/schema_parquet.go:382-399` — `ContractEventOutputParquet` declares `OperationID int64` as the last field.
- `internal/transform/parquet_converter.go:425-441` — `ToParquet()` maps all fields from `TransactionHash` through `ContractEventXDR` but stops before `OperationID`. The field is simply missing from the struct literal.
- `internal/transform/contract_events_test.go:74,126` — Tests expect `null.IntFrom(131090201534533633)` and `null.IntFrom(131090201534537729)` for operation-scoped events, confirming the JSON transform produces real operation IDs.
- `internal/transform/parquet_converter.go:155,281` — Other `ToParquet()` methods (for `OperationOutput` and `EffectOutput`) correctly include `OperationID` in their struct literals, confirming this is an isolated omission in the contract-event converter.

### Findings

The bug is confirmed. `ContractEventOutput.ToParquet()` omits the `OperationID` field assignment. Every Parquet row will contain `operation_id = 0` regardless of whether the event is operation-scoped or not. This has two downstream consequences:

1. **Broken joins**: Parquet consumers joining `history_contract_events.operation_id` to `history_operations.id` will fail to match any operation (since TOID `0` is not a valid operation ID), or will join to the wrong row if a zero-ID sentinel exists.
2. **Lost event classification**: The distinction between transaction-level events (`OperationID` should be null/0) and operation-scoped events (`OperationID` should be a valid TOID) is erased in Parquet output. All events appear to be transaction-level.

The JSON output path is unaffected — only the Parquet serialization is broken.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go` (append a new test) or create `internal/transform/data_integrity_poc_test.go`
- **Setup**: Use existing test fixtures from `makeContractEventTestInput()` that produce operation-scoped events
- **Steps**: Call `TransformContractEvent()` to get the JSON output, verify `OperationID.Valid == true` and `OperationID.Int64 != 0` for an operation-scoped event, then call `.ToParquet()` on that same output
- **Assertion**: Assert that the Parquet struct's `OperationID` field equals `0` (demonstrating the bug). The JSON struct's `OperationID.Int64` value should be non-zero (e.g., `131090201534537729`), but after `ToParquet()` the Parquet struct's `OperationID` will be `0`. Cast the `ToParquet()` return value to `ContractEventOutputParquet` and compare `parquetOutput.OperationID` against `jsonOutput.OperationID.Int64`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractEventParquetDropsOperationID"
**Test Language**: Go

### Demonstration

The test constructs a `ContractEventOutput` with a valid, non-zero `OperationID` (131090201534537729) representing an operation-scoped contract event, then calls `ToParquet()`. The resulting `ContractEventOutputParquet` struct has `OperationID = 0` because the converter at `parquet_converter.go:425-441` omits the `OperationID` field assignment entirely. This proves that every Parquet row silently loses its operation ID, breaking downstream joins and erasing event classification.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/guregu/null"
)

func TestContractEventParquetDropsOperationID(t *testing.T) {
	// Construct a ContractEventOutput with a non-zero, valid OperationID
	// simulating an operation-scoped contract event.
	operationID := int64(131090201534537729)
	jsonOutput := ContractEventOutput{
		TransactionHash:          "abc123",
		TransactionID:            131090201534537728,
		Successful:               true,
		LedgerSequence:           30521816,
		InSuccessfulContractCall: true,
		ContractId:               "CAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABSC4",
		Type:                     1,
		TypeString:               "ContractEventTypeContract",
		ContractEventXDR:         "deadbeef",
		OperationID:              null.IntFrom(operationID),
	}

	// Verify the JSON struct has the correct OperationID
	if !jsonOutput.OperationID.Valid {
		t.Fatal("precondition failed: JSON output OperationID should be valid")
	}
	if jsonOutput.OperationID.Int64 != operationID {
		t.Fatalf("precondition failed: JSON output OperationID = %d, want %d",
			jsonOutput.OperationID.Int64, operationID)
	}

	// Convert to Parquet
	parquetRaw := jsonOutput.ToParquet()
	parquetOutput, ok := parquetRaw.(ContractEventOutputParquet)
	if !ok {
		t.Fatal("ToParquet() did not return ContractEventOutputParquet")
	}

	// The bug: Parquet OperationID should match the JSON OperationID,
	// but the converter omits it, so it defaults to 0.
	if parquetOutput.OperationID != operationID {
		t.Errorf("OperationID corrupted in Parquet: got %d, want %d (non-zero operation ID lost)",
			parquetOutput.OperationID, operationID)
	}
}
```

### Test Output

```
=== RUN   TestContractEventParquetDropsOperationID
    data_integrity_poc_test.go:45: OperationID corrupted in Parquet: got 0, want 131090201534537729 (non-zero operation ID lost)
--- FAIL: TestContractEventParquetDropsOperationID (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.837s
```
