# H001: `TransformContractEvent()` double-counts contract events mirrored into diagnostics

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_contract_events` should contain one exported row per emitted event. If txmeta exposes the same contract event through both the canonical contract-event stream and the optional diagnostic-event stream, the transform should preserve only one copy or otherwise distinguish the two sources so downstream consumers do not count the same event twice.

## Mechanism

`TransformContractEvent()` appends every event from `TransactionEvents`, every event from `OperationEvents`, and every entry from `DiagnosticEvents` into one flat slice without any deduplication or source guard. The upstream SDK explicitly documents that diagnostic events may include contract events as well, so a txmeta producer that mirrors contract events into diagnostics will cause the ETL to emit duplicate rows with the same payload, contract ID, topics, data, and success flags.

## Trigger

Export a Soroban transaction whose txmeta includes a contract event in `OperationEvents` and also repeats that same event in `DiagnosticEvents` (the SDK documents this as configuration-dependent). The current transform will append both and produce two `history_contract_events` rows for one on-chain event.

## Target Code

- `internal/transform/contract_events.go:21-67` — `TransformContractEvent()` appends transaction, operation, and diagnostic streams into one slice with no dedupe.
- `internal/transform/contract_events.go:167-241` — `parseDiagnosticEvent()` normalizes all three sources into the same output shape, so mirrored events become indistinguishable duplicates.

## Evidence

The code path is straightforward: every `transactionEvents.TransactionEvents`, every `transactionEvents.OperationEvents[i]`, and every `transactionEvents.DiagnosticEvents` entry is appended. The upstream `ingest.LedgerTransaction.GetDiagnosticEvents()` documentation warns that diagnostic events "MAY include contract events as well" and that consumers should be careful not to double count them, but `TransformContractEvent()` does not apply any such guard.

## Anti-Evidence

If txmeta is generated without duplicating contract events into diagnostics, no duplicate rows appear. Some downstream users may also notice duplicates by joining on `contract_event_xdr`, but the exported table itself currently provides no source discriminator or dedupe.
