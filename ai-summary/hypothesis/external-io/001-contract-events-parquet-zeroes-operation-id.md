# H001: Contract event Parquet zeroes `operation_id`

**Date**: 2026-04-10
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_contract_events` is run with Parquet output, each operation-scoped contract event should preserve the same `operation_id` that appears in the JSON export. Transaction-scoped and diagnostic-only events should remain unset/null rather than being rewritten to a valid-looking zero.

## Mechanism

`TransformContractEvent()` explicitly computes a TOID-backed `operation_id` for `OperationEvents`, and both the JSON schema and Parquet schema declare that field. But `ContractEventOutput.ToParquet()` never assigns `OperationID`, so the Parquet writer serializes Go's zero value instead of the real TOID, turning operation-scoped rows into `operation_id = 0`.

## Trigger

Run `export_contract_events --write-parquet` on any ledger range containing a Soroban transaction with per-operation contract events. Compare the JSON row's non-zero `operation_id` with the Parquet row for the same event; the Parquet value should come out as `0`.

## Target Code

- `internal/transform/contract_events.go:41-55` — operation-scoped events get a real `OperationID`
- `internal/transform/schema.go:640-657` — JSON output includes `operation_id`
- `internal/transform/schema_parquet.go:382-399` — Parquet schema also declares `operation_id`
- `internal/transform/parquet_converter.go:425-441` — `ToParquet()` omits `OperationID`
- `cmd/export_contract_events.go:63-65` — this path writes the broken Parquet rows

## Evidence

`TransformContractEvent()` sets `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)` for operation events, and tests already expect populated IDs in `contract_events_test.go`. The Parquet struct has an `OperationID int64` field, but the converter never fills it, so every row gets the default zero.

## Anti-Evidence

JSON output is unaffected, and transaction-level diagnostic events legitimately have no operation ID. The corruption is confined to the Parquet export path and is easiest to demonstrate on operation-scoped events, where the correct value is definitely non-zero.
