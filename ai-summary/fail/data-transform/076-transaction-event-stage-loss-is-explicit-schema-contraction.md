# H076: transaction-event stage loss is an explicit schema contraction

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `history_contract_events` is meant to preserve CAP-67 unified-event semantics, transaction-level events should retain their `TransactionEvent.Stage` so `before_all_txs`, `after_tx`, and `after_all_txs` events remain distinguishable in the export.

## Mechanism

`transactionEvent2DiagnosticEvent()` converts `xdr.TransactionEvent` into `xdr.DiagnosticEvent` and explicitly discards `Stage` before the row is serialized. That initially looked like silent corruption because two stage-distinct transaction events with identical payloads would collapse to the same exported row shape.

## Trigger

Export a TransactionMetaV4 ledger containing two transaction events with the same `ContractEvent` payload but different `TransactionEvent.Stage` values, then compare the resulting `history_contract_events` rows.

## Target Code

- `internal/transform/contract_events.go:139-148` — `transactionEvent2DiagnosticEvent()` drops `TransactionEvent.Stage`.
- `internal/transform/schema.go:640-657` — `ContractEventOutput` has no field that can preserve transaction-event stage.

## Evidence

The code comment is explicit: "`TransactionEvent.Stage` is not preserved" and "`stellar-etl/hubble does not need to record the stage/ordering of events`". The transform therefore knowingly erases source metadata that exists in CAP-67 transaction events.

## Anti-Evidence

The same code comment also states that stage/ordering is intentionally out of scope for this export, and the schema contains no destination field for it. That makes this look like a deliberate flattening choice rather than an accidental wrong-value bug under the current repository contract.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The transform explicitly documents that stage/ordering is intentionally not part of the `history_contract_events` schema, and the output type has no column for it. This is a design contraction the codebase already acknowledges, not a fresh contradiction between the implemented transform and the current export contract.

### Lesson Learned

When a suspected omission is called out directly in code comments and the schema has no place to carry the missing source field, treat it as a schema-contract decision unless you can prove another part of the repository promises that field should be exported.
