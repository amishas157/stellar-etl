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

The production comment says the quiet part out loud: “`TransactionEvent.Stage` is not preserved”. Upstream XDR defines three distinct `TransactionEventStage` values, so the ETL is knowingly flattening semantically different events into the same output shape.

## Anti-Evidence

The exporter still preserves row order within a single in-memory slice, so a consumer that relies on positional ordering inside one JSON file might infer some sequencing. That is not durable schema-level data, and Parquet consumers lose even that weak signal.
