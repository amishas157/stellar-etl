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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-integrity/003-contract-event-parquet-operation-id-dropped.md.gh-published`, `ai-summary/success/data-transform/002-contract-event-parquet-operation-id-dropped.md.gh-published`, `ai-summary/success/external-io/001-contract-event-parquet-operation-id-dropped.md.gh-published`
**Failed At**: reviewer

### Trace Summary

The hypothesis correctly identifies that `ContractEventOutput.ToParquet()` (parquet_converter.go:425-442) constructs `ContractEventOutputParquet` without copying `ceo.OperationID`, causing the parquet `operation_id` column to default to `0` for operation-scoped events while JSON retains the correct TOID. The bug is real and the code trace confirms it. However, this exact finding has already been confirmed, PoC-verified, and published under three separate subsystems.

### Code Paths Examined

- `internal/transform/contract_events.go:42-54` — operation-scoped events correctly set `OperationID = null.IntFrom(operationID)`
- `internal/transform/schema.go:656` — `OperationID null.Int` present in JSON struct
- `internal/transform/schema_parquet.go:398` — `OperationID int64` present in parquet struct
- `internal/transform/parquet_converter.go:425-442` — `ToParquet()` omits `OperationID` from struct literal, confirmed missing

### Why It Failed

This is a duplicate of an already-confirmed and published finding. The identical root cause (missing `OperationID` assignment in `ContractEventOutput.ToParquet()`) has been independently discovered, PoC-verified, and published under three subsystems: `data-integrity` (003), `data-transform` (002), and `external-io` (001). The only difference is subsystem scoping — this hypothesis targets `cli-commands` while the prior findings were filed under other subsystems, but the underlying bug in `internal/transform/parquet_converter.go` is the same.

### Lesson Learned

Contract event parquet operation_id drop is one of the most frequently rediscovered findings in this codebase. Future hypotheses about parquet field omissions should check success directories across ALL subsystems, not just the hypothesis's own subsystem, since the transform layer code is shared.
