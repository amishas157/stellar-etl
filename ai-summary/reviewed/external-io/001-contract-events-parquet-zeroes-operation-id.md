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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full lifecycle of `OperationID` from TOID computation in `TransformContractEvent()` through to Parquet serialization. The transform function correctly computes and assigns a TOID-based `OperationID` via `null.IntFrom(operationID)` at `contract_events.go:52` for operation-scoped events. The JSON schema (`ContractEventOutput`) carries this value through to JSON output correctly. However, `ContractEventOutput.ToParquet()` at `parquet_converter.go:425-441` constructs a `ContractEventOutputParquet` struct that assigns every field EXCEPT `OperationID`, leaving it at Go's zero value (`0`). This is a clear copy-paste omission — sibling `ToParquet()` methods for `OperationOutput` (line 155) and `EffectOutput` (line 281) both correctly assign their `OperationID` fields.

### Code Paths Examined

- `internal/transform/contract_events.go:42-55` — Confirmed: operation-scoped events loop computes TOID and assigns `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)`
- `internal/transform/contract_events.go:167-242` — `parseDiagnosticEvent()` builds `ContractEventOutput` without `OperationID`; the caller in the OperationEvents loop adds it after
- `internal/transform/schema.go:641-657` — `ContractEventOutput` struct has `OperationID null.Int` as the last field
- `internal/transform/schema_parquet.go:383-399` — `ContractEventOutputParquet` struct has `OperationID int64` as the last field
- `internal/transform/parquet_converter.go:425-441` — `ToParquet()` maps 14 of 15 fields; `OperationID` is the one omitted
- `internal/transform/parquet_converter.go:147-161` — `OperationOutput.ToParquet()` correctly assigns `OperationID` (line 155), proving the pattern works in sibling code
- `internal/transform/parquet_converter.go:277-290` — `EffectOutput.ToParquet()` correctly assigns `OperationID` (line 281)
- `cmd/export_contract_events.go:50-51` — When `WriteParquet` is true, contract events are appended to `transformedEvents` and passed to `WriteParquet()`

### Findings

The bug is a straightforward omission. The `ContractEventOutput.ToParquet()` method constructs a `ContractEventOutputParquet` literal with 14 field assignments but skips the 15th field (`OperationID`). Since `ContractEventOutputParquet.OperationID` is `int64`, Go initializes it to `0`. This means every Parquet row — including operation-scoped events that have a valid, non-zero TOID — will report `operation_id = 0`.

The type mismatch between `null.Int` (JSON struct) and `int64` (Parquet struct) may explain why the field was omitted: the converter would need `ceo.OperationID.Int64` to extract the underlying value. But this is the same pattern used elsewhere (e.g., `TradeOutput.ToParquet()` uses `to.RoundingSlippage.Int64` at line 272).

Impact: Any downstream system consuming Parquet contract event data (e.g., BigQuery analytics) cannot join operation-scoped events back to their parent operations, since the join key is always zero. The JSON export path is unaffected.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (if it exists) or `internal/transform/contract_events_test.go`
- **Setup**: Use the existing test fixtures that produce `ContractEventOutput` with a non-zero `OperationID` (the tests in `contract_events_test.go` already create operation-scoped events)
- **Steps**: Call `TransformContractEvent()` on a transaction with operation-scoped events, then call `.ToParquet()` on the resulting `ContractEventOutput` that has a non-zero `OperationID`
- **Assertion**: Assert that the returned `ContractEventOutputParquet` has `OperationID` equal to the expected TOID value (not zero). Currently this will fail, confirming the bug. The fix is to add `OperationID: ceo.OperationID.Int64,` to the struct literal in `ToParquet()`.
