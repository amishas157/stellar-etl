# H003: Contract-event export discards `TransactionEvent.Stage`

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For protocol 23+ transaction-level events, `export_contract_events` should preserve the XDR `TransactionEvent.Stage` value so downstream analytics can distinguish events that happened `before_all_txs`, `after_tx`, and `after_all_txs`. Two otherwise identical fee/refund events emitted at different stages should remain distinguishable in exported rows.

## Mechanism

`TransformContractEvent()` converts each `xdr.TransactionEvent` into a synthetic `xdr.DiagnosticEvent` via `transactionEvent2DiagnosticEvent()`. That helper explicitly drops the `Stage` field, and neither `ContractEventOutput` nor `ContractEventOutputParquet` has any replacement column, so the stage information is lost before both JSON serialization and the `contract_event_xdr` base64 payload are generated.

## Trigger

Run `export_contract_events` on a protocol-23+ ledger whose transaction metadata contains transaction-level fee/refund events at multiple stages, especially two otherwise identical events that differ only by `TransactionEvent.Stage`. The exported rows will no longer indicate which stage each event came from.

## Target Code

- `internal/transform/contract_events.go:transactionEvent2DiagnosticEvent:139-149` — explicitly converts `TransactionEvent` to `DiagnosticEvent` and drops `Stage`
- `internal/transform/contract_events.go:TransformContractEvent:31-38` — routes transaction-level events through the lossy conversion
- `internal/transform/schema.go:ContractEventOutput:641-657` — JSON schema has no stage field
- `internal/transform/schema_parquet.go:ContractEventOutputParquet:383-399` — Parquet schema also has no stage field

## Evidence

The production comment says the quiet part out loud: "`TransactionEvent.Stage` is not preserved". Upstream XDR defines three distinct `TransactionEventStage` values, so the ETL is knowingly flattening semantically different events into the same output shape.

## Anti-Evidence

The exporter still preserves row order within a single in-memory slice, so a consumer that relies on positional ordering inside one JSON file might infer some sequencing. That is not durable schema-level data, and Parquet consumers lose even that weak signal.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full path from `TransformContractEvent()` through `transactionEvent2DiagnosticEvent()` at lines 142-149 of `contract_events.go`. The function converts `xdr.TransactionEvent` (which contains a `Stage` field of type `TransactionEventStage` with three enum values: `BeforeAllTxs=0`, `AfterTx=1`, `AfterAllTxs=2`) into an `xdr.DiagnosticEvent` which lacks a `Stage` field. The code comment at lines 140-141 explicitly states this is intentional: "stellar-etl/hubble does not need to record the stage/ordering of events."

### Code Paths Examined

- `internal/transform/contract_events.go:transactionEvent2DiagnosticEvent:139-149` — Explicit comment documents the intentional design decision to drop Stage
- `internal/transform/contract_events.go:TransformContractEvent:31-38` — Routes transaction-level events through the conversion; no stage is extracted before or after
- `internal/transform/schema.go:ContractEventOutput:641-657` — Confirmed no stage field in JSON schema
- `internal/transform/schema_parquet.go:ContractEventOutputParquet:383-399` — Confirmed no stage field in Parquet schema
- `go-stellar-sdk/xdr/xdr_generated.go:TransactionEvent` — Confirmed upstream struct has `Stage TransactionEventStage` field with 3 enum values
- `go-stellar-sdk/xdr/xdr_generated.go:TransactionEventStage` — Confirmed enum: BeforeAllTxs(0), AfterTx(1), AfterAllTxs(2)

### Why It Failed

This is **working-as-designed behavior**, not a bug. The code comment at line 140-141 explicitly documents the design decision: "Note that the TransactionEvent.Stage is not preserved when changed to a DiagnosticEvent — stellar-etl/hubble does not need to record the stage/ordering of events." The hypothesis itself acknowledges this in the Evidence section ("says the quiet part out loud"), but then frames it as a bug anyway.

The "expected behavior" stated in the hypothesis — that Stage should be preserved — contradicts the developers' explicitly documented design intent. Whether this design decision is optimal is a product question, not a data correctness bug. The code is producing exactly the output the developers intended.

### Lesson Learned

When a code comment explicitly documents that a field is intentionally omitted, the omission is a design decision, not a bug — even if the omitted data could theoretically be useful to downstream consumers. Hypotheses that acknowledge intentional design decisions but reframe them as bugs should be rejected as working-as-designed.
