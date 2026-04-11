# H004: Contract-event export can double-count events already present in diagnostics

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_contract_events` should emit each underlying contract event once, or it should preserve enough provenance to let downstream consumers distinguish “operation event” rows from “diagnostic mirror” rows without double-counting. A single emitted contract event should not become two history rows merely because the metadata also exposes it through the diagnostic stream.

## Mechanism

`TransformContractEvent()` blindly appends `TransactionEvents`, `OperationEvents`, and `DiagnosticEvents` into one output slice. Upstream `GetDiagnosticEvents()` explicitly warns that, for smart-contract transactions, diagnostic events **may include contract events as well** and callers should avoid double-counting them; the ETL does not deduplicate or tag the source stream, so mirrored events become duplicate exported rows.

## Trigger

Run `export_contract_events` on a smart-contract transaction whose transaction meta includes contract events inside `DiagnosticEvents` in addition to the normal contract-event stream. The exporter will append both copies and overstate the number of contract events for that transaction.

## Target Code

- `internal/transform/contract_events.go:TransformContractEvent:21-65` — concatenates transaction, operation, and diagnostic event arrays with no dedupe or source column
- `internal/transform/contract_events.go:parseDiagnosticEvent:167-241` — normalizes all three streams to the same `ContractEventOutput` shape
- `internal/transform/contract_events_test.go:58-145` — current fixture already encodes the duplicated-row pattern for v3/v4 metadata

## Evidence

`go doc -src github.com/stellar/go-stellar-sdk/xdr.TransactionMeta.GetDiagnosticEvents` and the ingest wrapper both document that `DiagnosticEvents` may include contract events and that callers “should be careful not to double count diagnostic events and contract events in that case”. The ETL currently does the exact opposite by unioning both streams into the same table.

## Anti-Evidence

If a node or metadata configuration emits strictly non-contract diagnostic events, no duplication occurs. The bug depends on the documented configuration where diagnostics mirror contract events, but that configuration is explicitly supported upstream rather than theoretical.
