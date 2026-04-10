# H002: Contract-event parquet export zeroes `operation_id`

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For operation-scoped contract events, parquet output should preserve the same `operation_id` TOID that JSON output already carries. In the repository's contract-event fixtures, rows whose JSON output has `operation_id=131090201534533633` or `131090201534537729` should produce parquet rows with those exact values.

## Mechanism

`TransformContractEvent()` explicitly computes and assigns `OperationID` for `OperationEvents`, and `ContractEventOutputParquet` even defines an `operation_id` column. But `ContractEventOutput.ToParquet()` never assigns that field, so parquet rows keep the struct zero value `0` instead of the real operation TOID. This silently rewrites operation-scoped events into a misleading “operation 0” shape.

## Trigger

Run `export_contract_events --write-parquet` on any ledger containing Soroban operation events. JSON output shows a non-null `operation_id` for those rows, while parquet output emits `0`.

## Target Code

- `internal/transform/contract_events.go:TransformContractEvent:41-55` — assigns `OperationID` for operation-level events
- `internal/transform/schema.go:ContractEventOutput:640-657` — JSON schema stores `OperationID` as `null.Int`
- `internal/transform/contract_events_test.go:58-145` — existing expected outputs include both null and non-null `OperationID` values
- `internal/transform/schema_parquet.go:ContractEventOutputParquet:382-398` — parquet schema includes `OperationID int64`
- `internal/transform/parquet_converter.go:ContractEventOutput.ToParquet:425-441` — converter omits `OperationID`
- `cmd/export_contract_events.go:contractEventsCmd.Run:50-65` — command serializes the broken parquet rows

## Evidence

This is a classic one-field copy omission: the parquet schema already has `operation_id`, and the JSON transform already computes it. The only thing missing is the field assignment in `ToParquet()`.

## Anti-Evidence

Transaction-level and diagnostic-only events legitimately have no operation ID, so the parquet schema also has a broader nullability problem. But even ignoring that larger design issue, operation-scoped events with real non-zero IDs are concretely wrong today because they are always exported as `0`.
