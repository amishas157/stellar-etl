# H003: Contract-event parquet export zeroes `operation_id` for operation-scoped events

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For contract events emitted from a specific operation, both JSON and parquet exports should contain the same non-null `operation_id` so downstream analytics can attribute each event to the correct operation. Only transaction-level or diagnostic events without an operation scope should have a null/empty `operation_id`.

## Mechanism

`TransformContractEvent` explicitly sets `OperationID` for entries produced from `transactionEvents.OperationEvents`. `ContractEventOutputParquet` also defines an `operation_id` column. But `ContractEventOutput.ToParquet()` never copies `ceo.OperationID` into that struct, so parquet rows default to `0` for operation-scoped events. This creates silent divergence where JSON has the correct TOID and parquet reports a plausible-but-wrong zero value.

## Trigger

Run `export_contract_events --write-parquet` on any ledger range containing Soroban events emitted by a specific operation. Compare the JSON and parquet rows for those events: JSON shows a non-null `operation_id`, while parquet stores `0`.

## Target Code

- `internal/transform/contract_events.go:41-55` — operation-scoped contract events get a computed `OperationID`
- `internal/transform/schema.go:640-657` — JSON schema includes nullable `operation_id`
- `internal/transform/schema_parquet.go:382-399` — parquet schema includes `operation_id`
- `internal/transform/parquet_converter.go:425-442` — parquet conversion omits `OperationID`
- `cmd/export_contract_events.go:42-65` — command writes the same transformed events to JSON and parquet

## Evidence

The transform path assigns `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)` for operation events, but the parquet converter returns `ContractEventOutputParquet{...}` without an `OperationID:` assignment. Since `ContractEventOutputParquet.OperationID` is plain `int64`, omitted assignment becomes `0` for every parquet row.

## Anti-Evidence

Transaction-level events legitimately have no operation ID, so rows from `TransactionEvents` or `DiagnosticEvents` are unaffected. The corruption is limited to the operation-scoped subset where JSON proves the value exists.
