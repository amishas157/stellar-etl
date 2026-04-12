# H002: Transaction-level contract events lose `stage` and become indistinguishable

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `TransactionMetaV4.Events`, `export_contract_events` should preserve the
transaction-event stage so downstream consumers can distinguish
`BEFORE_ALL_TXS`, `AFTER_TX`, and `AFTER_ALL_TXS` rows. At minimum, the export
should retain enough raw XDR to reconstruct the original `TransactionEvent`,
including its `stage` field.

## Mechanism

`TransformContractEvent()` converts every `TransactionEvent` into a synthetic
`DiagnosticEvent` before parsing it. That helper explicitly drops
`TransactionEvent.Stage`, the output schema has no stage column, and
`contract_event_xdr` is then marshaled from the synthetic `DiagnosticEvent`
instead of the original `TransactionEvent`. As a result, transaction-level
events from different phases collapse into the same exported shape.

## Trigger

1. Export contract events from a Protocol-25+ ledger whose `TransactionMetaV4`
   contains transaction-level events in `events<>` (for example fee-payment or
   other transaction-scoped Soroban events).
2. Include at least two events at different stages such as
   `BEFORE_ALL_TXS` and `AFTER_TX`.
3. Inspect the resulting `history_contract_events` rows: there is no exported
   stage field, and `contract_event_xdr` cannot recover it because the ETL
   serialized a synthetic `DiagnosticEvent` wrapper instead.

## Target Code

- `internal/transform/contract_events.go:31-38` — transaction-level events are
  first rewritten before export
- `internal/transform/contract_events.go:139-149` — `transactionEvent2DiagnosticEvent`
  explicitly drops `TransactionEvent.Stage`
- `internal/transform/contract_events.go:167-223` — `parseDiagnosticEvent`
  marshals the synthetic wrapper into `contract_event_xdr`
- `internal/transform/schema.go:640-656` — exported schema has no stage column
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:18173-18223`
  — `TransactionEvent` carries a real `stage` field in XDR
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:18259-18262`
  — `TransactionMetaV4.events<>` is the transaction-level event stream

## Evidence

The local comment already acknowledges the loss: "`TransactionEvent.Stage` is
not preserved." Unlike earlier aliasing false positives, this field does exist
in current generated XDR and is not exported anywhere else, so the dropped
information is real and unrecoverable from the row shape the ETL emits.

## Anti-Evidence

The code comments and tests suggest this flattening is deliberate, so review
should confirm whether `history_contract_events` is supposed to be a lossy
analytics surface. If the table is intentionally only a coarse event stream, the
reviewer may treat stage loss as a design trade-off rather than a bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/fail/export-pipeline/summary.md` entry 011
**Failed At**: reviewer

### Trace Summary

The hypothesis claims `TransactionEvent.Stage` is dropped during contract event export, causing events from different stages to become indistinguishable. This exact finding was previously investigated and rejected as entry 011 in the `export-pipeline` fail summary. The code at `contract_events.go:140-141` contains an explicit comment: "Note that the TransactionEvent.Stage is not preserved when changed to a DiagnosticEvent / stellar-etl/hubble does not need to record the stage/ordering of events." This is a documented, intentional design decision.

### Code Paths Examined

- `internal/transform/contract_events.go:139-149` — `transactionEvent2DiagnosticEvent` drops `Stage` with an explicit comment acknowledging and justifying the omission
- `internal/transform/contract_events.go:31-38` — Transaction events loop through the conversion
- `internal/transform/schema.go:640-656` — `ContractEventOutput` has no stage field, consistent with the intentional omission

### Why It Failed

This is a duplicate of a previously investigated and rejected hypothesis (`export-pipeline` fail summary entry 011). The prior investigation concluded that the `TransactionEvent.Stage` omission is an explicitly documented product decision — the code comment at lines 140-141 states "stellar-etl/hubble does not need to record the stage/ordering of events." An intentional, documented design trade-off is not a data integrity bug.

### Lesson Learned

Always check fail summaries across ALL subsystems (not just the target subsystem) for prior investigations of the same code path. Cross-subsystem duplicates are common when a data-integrity hypothesis targets the same mechanism previously reviewed under export-pipeline.
