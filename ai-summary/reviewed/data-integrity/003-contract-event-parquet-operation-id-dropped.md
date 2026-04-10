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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `TransformContractEvent()` function in `contract_events.go` correctly populates `OperationID` via `null.IntFrom(operationID)` at line 52 for all events emitted from `transactionEvents.OperationEvents`. The `ContractEventOutputParquet` struct in `schema_parquet.go:398` declares the `OperationID int64` Parquet column. However, the `ToParquet()` method in `parquet_converter.go:425-441` constructs the Parquet struct literal with 14 fields but omits `OperationID`, causing it to default to `0` (Go zero value for `int64`). The established pattern for converting `null.Int` fields to Parquet `int64` is `.Int64` (e.g., `to.MinAccountSequence.Int64` at line 84, `ao.SequenceLedger.Int64` at line 112), confirming the omission is unintentional.

### Code Paths Examined

- `internal/transform/contract_events.go:42-54` — Confirmed `OperationID` is set to a valid TOID for operation-scoped events via `null.IntFrom(operationID)`
- `internal/transform/schema.go:656` — `OperationID null.Int` field is the last field in `ContractEventOutput`, making it easy to miss during copy-paste of the converter
- `internal/transform/schema_parquet.go:398` — `OperationID int64` is the last field in `ContractEventOutputParquet`, confirming the Parquet schema expects this column
- `internal/transform/parquet_converter.go:425-441` — `ToParquet()` maps 14 of 15 fields; `OperationID` is the sole omission
- `internal/transform/parquet_converter.go:84-86,112-113,266-267,269,272` — Established pattern for `null.Int` → `int64` conversion uses `.Int64` accessor

### Findings

This is a Pattern 2 (Parquet field mapping) bug. The `ToParquet()` method for `ContractEventOutput` copies every field except `OperationID`. The omitted field is the last one in both the JSON and Parquet struct definitions — a classic copy-paste oversight where the final field was never added. Every operation-scoped contract event exported to Parquet silently loses its operation linkage, making it impossible for downstream consumers to join contract events to their originating operations using the Parquet export path. JSON exports are unaffected.

The fix is a one-line addition to `parquet_converter.go:441` (before the closing brace):
```go
OperationID:              ceo.OperationID.Int64,
```

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or `internal/transform/contract_events_test.go` if no converter tests exist)
- **Setup**: Create a `ContractEventOutput` with `OperationID` set to a non-zero `null.IntFrom(someValue)`, e.g. `null.IntFrom(12345)`
- **Steps**: Call `.ToParquet()` on the struct and cast the result to `ContractEventOutputParquet`
- **Assertion**: Assert that the returned `ContractEventOutputParquet.OperationID` equals the input value (12345), not 0
