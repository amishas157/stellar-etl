# H003: Parquet contract-event export drops non-null `operation_id`

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For operation-scoped contract events, the Parquet row should preserve the same `operation_id` that the JSON row carries. If an event is emitted from operation index 1, the exported Parquet row should contain the corresponding TOID instead of a default value.

## Mechanism

`TransformContractEvent()` correctly sets `ContractEventOutput.OperationID` for operation events, and `ContractEventOutputParquet` defines an `operation_id` column, but `ContractEventOutput.ToParquet()` never copies that field. The Parquet struct therefore serializes the zero value instead of the populated TOID.

## Trigger

Export contract events to Parquet for any transaction with entries in `transactionEvents.OperationEvents`.

## Target Code

- `internal/transform/contract_events.go:42-54` — assigns `OperationID` for operation-scoped events
- `internal/transform/schema.go:640-657` — JSON schema stores `operation_id` as `null.Int`
- `internal/transform/schema_parquet.go:383-399` — Parquet schema declares `operation_id`
- `internal/transform/parquet_converter.go:425-441` — omits the `OperationID` assignment

## Evidence

The JSON path explicitly sets `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)`, and the Parquet schema includes an `OperationID int64` field. The converter returns `ContractEventOutputParquet{...}` without ever setting `OperationID`, so every Parquet row falls back to the zero value.

## Anti-Evidence

JSON exports remain correct, and transaction-level or diagnostic-only events are expected to have no operation id. The corruption is limited to the Parquet path.
