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

Some contract-event rows legitimately have no operation context, so a nullable representation is required somewhere in the pipeline. But that makes the current omission worse, not safer: Parquet now conflates “no operation” with “operation-scoped event whose ID was never serialized.”
