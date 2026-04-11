# H001: Contract-event parquet export drops every operation-scoped TOID

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_contract_events --write-parquet` processes an operation-scoped contract event, the parquet row's `operation_id` should contain the same TOID that the JSON row emits and that `history_operations.id` uses for the parent operation.

## Mechanism

`TransformContractEvent()` correctly computes an operation TOID for rows coming from `transactionEvents.OperationEvents` and stores it in `ContractEventOutput.OperationID`. But `ContractEventOutput.ToParquet()` never copies that field into `ContractEventOutputParquet`, so parquet-go writes the zero value `0` for every row. The parquet export therefore silently replaces real operation TOIDs with a plausible-looking integer that no longer joins back to the originating operation.

## Trigger

Run `export_contract_events --write-parquet` on any ledger containing at least one operation-scoped Soroban contract event.

## Target Code

- `internal/transform/contract_events.go:41-54` — operation-scoped events get `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)`.
- `internal/transform/schema.go:641-657` — JSON schema models `operation_id` as `null.Int`.
- `internal/transform/schema_parquet.go:382-399` — parquet schema declares a physical `operation_id` column.
- `internal/transform/parquet_converter.go:425-441` — `ContractEventOutput.ToParquet()` omits `OperationID` entirely.
- `cmd/export_contract_events.go:50-65` — contract-event rows are appended and written through the parquet converter when `--write-parquet` is enabled.

## Evidence

The operation-event branch explicitly computes the TOID with `toid.New(..., int32(i)+1).ToInt64()` and stores it on the JSON struct, but the parquet converter returns a struct literal with no `OperationID:` assignment. Because `ContractEventOutputParquet.OperationID` is an `int64`, the emitted parquet value becomes `0` for every row regardless of the source event.

## Anti-Evidence

The JSON export path is correct, so this corruption is limited to parquet output. Downstream systems that consume only JSON would not observe the bug.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformContractEvent` (contract_events.go:42-55) iterates `transactionEvents.OperationEvents`, computes a TOID via `toid.New(ledger, txIndex, opIndex+1).ToInt64()`, and stores it in `ContractEventOutput.OperationID` as `null.IntFrom(operationID)`. The `ContractEventOutput` struct (schema.go:656) declares `OperationID null.Int`. The parquet schema (schema_parquet.go:398) declares a corresponding `OperationID int64` column. However, `ContractEventOutput.ToParquet()` (parquet_converter.go:425-442) constructs the `ContractEventOutputParquet` literal with 14 fields but omits `OperationID`, causing it to default to Go's zero value `0` for every row.

### Code Paths Examined

- `internal/transform/contract_events.go:42-55` — operation-scoped events correctly compute TOID and assign `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)`
- `internal/transform/contract_events.go:167-242` — `parseDiagnosticEvent` constructs `ContractEventOutput` without setting `OperationID` (it is set post-hoc by the caller for operation events only)
- `internal/transform/schema.go:641-657` — `ContractEventOutput.OperationID` is `null.Int` (wraps `sql.NullInt64` with `Int64` and `Valid` fields)
- `internal/transform/schema_parquet.go:382-399` — `ContractEventOutputParquet.OperationID` is `int64` (non-nullable, defaults to 0)
- `internal/transform/parquet_converter.go:425-442` — `ToParquet()` struct literal has 14 field assignments but `OperationID` is absent; confirmed by checking every other `ToParquet()` method in the file (e.g., line 155 for operations, line 281 for effects) which DO include `OperationID`

### Findings

The omission is clearly a copy-paste oversight. Every other parquet converter that has an `OperationID` field includes it in the struct literal. The contract event converter is the only one that omits it. The fix is a single line addition: `OperationID: ceo.OperationID.Int64,` in the struct literal at parquet_converter.go:441.

For transaction-scoped and diagnostic events, `OperationID` remains at its zero value (`null.Int{}` where `Int64 = 0, Valid = false`), so parquet output of `0` is acceptable for those rows (parquet `int64` cannot represent null). The data loss is specifically for operation-scoped events where the real TOID is computed but never transferred to parquet output.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go` (if it exists) or `internal/transform/parquet_converter_test.go`
- **Setup**: Create a mock `ingest.LedgerTransaction` with at least one operation-scoped contract event (populate `transactionEvents.OperationEvents`)
- **Steps**: Call `TransformContractEvent(tx, lhe)`, then call `.ToParquet()` on each returned `ContractEventOutput`
- **Assertion**: For operation-scoped event rows, assert that the parquet struct's `OperationID` field equals the JSON struct's `OperationID.Int64` and is non-zero. Currently the parquet value will be `0` while the JSON value is a valid TOID.
