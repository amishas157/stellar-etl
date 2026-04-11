# H004: Contract-event export can double-count events already present in diagnostics

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_contract_events` should emit each underlying contract event once, or it should preserve enough provenance to let downstream consumers distinguish "operation event" rows from "diagnostic mirror" rows without double-counting. A single emitted contract event should not become two history rows merely because the metadata also exposes it through the diagnostic stream.

## Mechanism

`TransformContractEvent()` blindly appends `TransactionEvents`, `OperationEvents`, and `DiagnosticEvents` into one output slice. Upstream `GetDiagnosticEvents()` explicitly warns that, for smart-contract transactions, diagnostic events **may include contract events as well** and callers should avoid double-counting them; the ETL does not deduplicate or tag the source stream, so mirrored events become duplicate exported rows.

## Trigger

Run `export_contract_events` on a smart-contract transaction whose transaction meta includes contract events inside `DiagnosticEvents` in addition to the normal contract-event stream. The exporter will append both copies and overstate the number of contract events for that transaction.

## Target Code

- `internal/transform/contract_events.go:TransformContractEvent:21-65` — concatenates transaction, operation, and diagnostic event arrays with no dedupe or source column
- `internal/transform/contract_events.go:parseDiagnosticEvent:167-241` — normalizes all three streams to the same `ContractEventOutput` shape
- `internal/transform/contract_events_test.go:58-145` — current fixture already encodes the duplicated-row pattern for v3/v4 metadata

## Evidence

`go doc -src github.com/stellar/go-stellar-sdk/xdr.TransactionMeta.GetDiagnosticEvents` and the ingest wrapper both document that `DiagnosticEvents` may include contract events and that callers "should be careful not to double count diagnostic events and contract events in that case". The ETL currently does the exact opposite by unioning both streams into the same table.

## Anti-Evidence

If a node or metadata configuration emits strictly non-contract diagnostic events, no duplication occurs. The bug depends on the documented configuration where diagnostics mirror contract events, but that configuration is explicitly supported upstream rather than theoretical.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `GetTransactionEvents()` in the upstream SDK through `TransformContractEvent()`. For V3 metadata (Soroban transactions), the SDK's `GetTransactionEvents()` populates `OperationEvents[0]` from `SorobanMeta.Events` and `DiagnosticEvents` from `SorobanMeta.DiagnosticEvents`. The SDK explicitly documents that diagnostic events "MAY include contract events as well" and warns callers to "be careful not to double count." The ETL ignores this warning and concatenates all arrays into a single output slice. The existing test fixture confirms this: it constructs identical events in both arrays and expects 2 output rows for the same underlying event, with differing `OperationID` values (one set, one null).

### Code Paths Examined

- `go-stellar-sdk/ingest/ledger_transaction.go:GetTransactionEvents:278-320` — For V3: populates `OperationEvents[0]` from `GetContractEvents()` (→ `SorobanMeta.Events`) and `DiagnosticEvents` from `GetDiagnosticEvents()` (→ `SorobanMeta.DiagnosticEvents`). For V4: populates all three arrays from raw XDR. No deduplication in either case.
- `go-stellar-sdk/xdr/transaction_meta.go:GetDiagnosticEvents:35-49` — Explicit SDK warning at lines 33-36: "the list of generated diagnostic events MAY include contract events...should be careful not to double count"
- `internal/transform/contract_events.go:TransformContractEvent:21-67` — Three sequential loops concatenate `TransactionEvents` (lines 31-39), `OperationEvents` (lines 42-56), and `DiagnosticEvents` (lines 58-65) into one slice with no dedup
- `internal/transform/contract_events_test.go:150-208` — V3 test fixture places the same ContractEvent (type=Diagnostic, ContractId=zero, topics=[true], data=true) in both `SorobanMeta.Events` and `SorobanMeta.DiagnosticEvents`
- `internal/transform/contract_events_test.go:58-92` — Expected V3 output: 2 rows for 1 underlying event — first row has `OperationID` set (from OperationEvents loop), second has null `OperationID` (from DiagnosticEvents loop)

### Findings

1. **V3 double-counting is confirmed and tested**: The test fixture creates identical event data in `SorobanMeta.Events` and `SorobanMeta.DiagnosticEvents`. `GetTransactionEvents()` returns both arrays unmodified. `TransformContractEvent()` processes both, producing 2 rows with differing `OperationID` for the same underlying event. This is exactly the pattern the SDK warns against.

2. **V4 overlap is plausible**: For V4 Soroban transactions, `DiagnosticEvents` may also mirror `Operations[i].Events` depending on node configuration. The ETL would produce duplicates in this case too. The V4 test fixture uses distinct event types across the three arrays, so V4 double-counting is not exercised by tests but is architecturally possible.

3. **No dedup key or source column**: `ContractEventOutput` has no field indicating which stream (TransactionEvents/OperationEvents/DiagnosticEvents) an event originated from. Downstream consumers in BigQuery cannot distinguish or deduplicate.

4. **Inconsistent OperationID on duplicates**: The same underlying event gets `OperationID` set when processed through the OperationEvents loop but null when processed through the DiagnosticEvents loop. This means the duplicate rows are not even identical — they differ in a semantically important field.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go`
- **Setup**: Construct a V3 `ingest.LedgerTransaction` where `SorobanMeta.Events` contains a contract event (type=Contract) and `SorobanMeta.DiagnosticEvents` contains a DiagnosticEvent wrapping an identical ContractEvent. This mirrors real-world behavior when diagnostic events include contract events.
- **Steps**: Call `TransformContractEvent(transaction, header)` and count the output rows.
- **Assertion**: Assert that the number of output rows equals the number of *unique* events, not the sum of all arrays. Alternatively, assert that each output row has a `source_stream` field or that duplicate events are tagged/filtered. The existing test at lines 58-92 already demonstrates the bug — it expects 2 rows for 1 unique event.
