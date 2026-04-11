# H002: Transaction-scoped contract events serialize NULL operation IDs as `0` in parquet

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For transaction-level contract events and diagnostic events that are not attached to a specific operation, parquet output should preserve `operation_id = NULL`, matching the JSON export and indicating that no operation TOID exists for that row.

## Mechanism

`TransformContractEvent()` appends transaction-level and diagnostic events without setting `OperationID`, so the source value is intentionally null. However, the parquet side models `operation_id` as a required `int64` instead of a nullable field, and the converter provides no validity check. As a result, rows with no operation context are serialized as `0`, manufacturing a fake TOID instead of preserving the absence of one.

## Trigger

Run `export_contract_events --write-parquet` on a ledger containing a transaction-scoped contract event or diagnostic event emitted outside `transactionEvents.OperationEvents`.

## Target Code

- `internal/transform/contract_events.go:30-39` — transaction-level events are appended without assigning `OperationID`.
- `internal/transform/contract_events.go:58-65` — diagnostic events are appended without assigning `OperationID`.
- `internal/transform/schema.go:641-657` — the source schema declares `OperationID null.Int`.
- `internal/transform/schema_parquet.go:382-399` — the parquet schema narrows that nullable field to plain `int64`.
- `internal/transform/parquet_converter.go:425-441` — parquet conversion does not preserve null validity.

## Evidence

The transform code uses `null.Int` specifically because some contract-event rows have no operation TOID. On the parquet side there is no nullable wrapper, no optional tag, and no branch that distinguishes `Valid=false` from `Int64==0`, so the semantic difference between "no operation" and "operation ID zero" is lost.

## Anti-Evidence

There is no real Stellar operation whose canonical TOID is `0`, so consumers may be able to recognize the bad rows after the fact. But the export itself is still wrong because it turns a null relationship into a numeric value.
