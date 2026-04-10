# H002: Contract-event parquet export zeroes `operation_id`

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For operation-scoped contract events, parquet output should preserve the same `operation_id` TOID that JSON output already carries. In the repository's contract-event fixtures, rows whose JSON output has `operation_id=131090201534533633` or `operation_id=131090201534537729` should produce parquet rows with those exact values.

## Mechanism

`TransformContractEvent()` explicitly computes and assigns `OperationID` for `OperationEvents`, and `ContractEventOutputParquet` even defines an `operation_id` column. But `ContractEventOutput.ToParquet()` never assigns that field, so parquet rows keep the struct zero value `0` instead of the real operation TOID. This silently rewrites operation-scoped events into a misleading "operation 0" shape.

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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full path from `TransformContractEvent()` through `parseDiagnosticEvent()` to the `ToParquet()` converter. The transform function correctly computes `OperationID` via `toid.New()` for operation-scoped events (line 50-52 of `contract_events.go`) and assigns it to the `ContractEventOutput.OperationID` field as `null.IntFrom(operationID)`. The JSON schema at `schema.go:656` carries this value as `null.Int`. The Parquet schema at `schema_parquet.go:398` declares `OperationID int64`. However, the `ToParquet()` method at `parquet_converter.go:425-442` constructs the `ContractEventOutputParquet` struct with 13 field assignments but omits `OperationID` entirely, leaving it at Go's zero value of `0`.

### Code Paths Examined

- `internal/transform/contract_events.go:TransformContractEvent:42-55` — iterates `OperationEvents`, computes `operationID` via `toid.New(ledgerSeq, txIndex, opIndex+1).ToInt64()`, assigns `parsedDiagnosticEvent.OperationID = null.IntFrom(operationID)`. Confirmed non-zero IDs are produced for operation-scoped events.
- `internal/transform/contract_events.go:parseDiagnosticEvent:167-242` — builds the base `ContractEventOutput` struct but does NOT set `OperationID` (it stays as zero-value `null.Int`). The caller (`TransformContractEvent`) sets it after the call for operation events only. Correct design.
- `internal/transform/schema.go:641-657` — `ContractEventOutput` struct defines `OperationID null.Int` at line 656. JSON marshalling correctly includes this field.
- `internal/transform/schema_parquet.go:382-399` — `ContractEventOutputParquet` struct defines `OperationID int64` at line 398. The column exists in the Parquet schema.
- `internal/transform/parquet_converter.go:425-442` — `ToParquet()` method constructs `ContractEventOutputParquet` with fields: TransactionHash, TransactionID, Successful, LedgerSequence, ClosedAt, InSuccessfulContractCall, ContractId, Type, TypeString, Topics, TopicsDecoded, Data, DataDecoded, ContractEventXDR. **`OperationID` is absent from this list.** The struct field defaults to `int64(0)`.

### Findings

The bug is a textbook Pattern 2 (Parquet field mapping copy-paste omission). All 14 fields of `ContractEventOutputParquet` are defined in the schema, but only 13 are assigned in `ToParquet()`. The missing assignment should be `OperationID: ceo.OperationID.Int64` (extracting the `int64` from the `null.Int` wrapper). For events without an operation ID (transaction-level and diagnostic events), `null.Int` zero-value's `.Int64` field is `0`, which matches the current (incorrect) behavior for all events — so the fix only changes output for operation-scoped events where the value is non-zero.

This causes every Parquet consumer that joins contract events to operations via `operation_id` to get zero matches for operation-scoped events, silently dropping the relationship between events and their triggering operations.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go`
- **Setup**: Use the existing test fixtures that produce `ContractEventOutput` with non-null `OperationID` values. The existing test at lines 58-145 already validates JSON output with specific `OperationID` values.
- **Steps**: Call `TransformContractEvent()` on the existing test fixture that produces operation-scoped events. Then call `.ToParquet()` on each result and inspect the `OperationID` field of the returned `ContractEventOutputParquet`.
- **Assertion**: Assert that for operation-scoped events, `parquetOutput.OperationID` equals `jsonOutput.OperationID.Int64` (non-zero). Currently it will be `0` for all events, confirming the bug.
