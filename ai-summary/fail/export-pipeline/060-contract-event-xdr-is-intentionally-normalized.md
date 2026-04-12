# H060: `contract_event_xdr` looked like a raw event blob, but it is intentionally normalized to `DiagnosticEvent`

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: raw-blob fidelity
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `contract_event_xdr` were meant to preserve the original XDR object for each emitted row, transaction-scoped events would retain their `TransactionEvent` wrapper (including stage), and operation-scoped events would retain their original `ContractEvent` bytes rather than an ETL-fabricated wrapper.

## Mechanism

The contract-event pipeline converts `TransactionEvent` and `ContractEvent` values into synthetic `xdr.DiagnosticEvent` wrappers and then marshals that wrapper into `contract_event_xdr`. That initially looked like silent raw-XDR corruption because the blob no longer reflects the protocol-native source object that produced the row.

## Trigger

Export any ledger containing transaction-scoped or operation-scoped contract events. The emitted `contract_event_xdr` will decode as a `DiagnosticEvent`, even when the original upstream object was a `TransactionEvent` or `ContractEvent`.

## Target Code

- `internal/transform/contract_events.go:139-158` — converts `TransactionEvent` and `ContractEvent` values into synthetic `DiagnosticEvent` wrappers
- `internal/transform/contract_events.go:161-166` — comments describe the normalized cross-family output contract
- `internal/transform/contract_events.go:219-238` — marshals the normalized `diagnosticEvent` into `contract_event_xdr`

## Evidence

The conversion helpers hard-code `InSuccessfulContractCall: true` and drop the original outer event type before `parseDiagnosticEvent()` serializes the blob. The resulting column therefore cannot be a byte-for-byte dump of the original upstream source object.

## Anti-Evidence

The same file explicitly documents that this exporter intentionally flattens transaction events, operation events, and diagnostic events into a common contract-event representation. The code comment also explicitly states that transaction-event stage/ordering is not preserved because the product does not need it.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is an intentional normalization layer, not an accidental raw-XDR drift. The subsystem explicitly chooses a common `DiagnosticEvent`-shaped representation for all event families, and `contract_event_xdr` is part of that normalized contract rather than a promise to preserve the exact upstream wrapper type.

### Lesson Learned

When a table deliberately flattens multiple upstream object families into one schema, a column ending in `_xdr` is not automatically a guarantee of protocol-native bytes. Check for local normalization comments before treating a wrapped blob as silent corruption.
