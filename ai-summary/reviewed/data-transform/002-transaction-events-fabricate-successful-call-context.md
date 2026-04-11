# H002: Transaction-level events fabricate `in_successful_contract_call=true`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Transaction-level events from `TransactionMetaV4.Events` should only export
`in_successful_contract_call=true` when that truth is actually available from
the source metadata. System/fee-payment transaction events that are not inside a
successful contract call should export `false` or otherwise remain distinct from
diagnostic events that really did occur inside a successful call.

## Mechanism

`transactionEvent2DiagnosticEvent()` fabricates a `DiagnosticEvent` and hard
codes `InSuccessfulContractCall: true` for every `xdr.TransactionEvent`.
`parseDiagnosticEvent()` then copies that invented value into the exported row.
Because `xdr.TransactionEvent` carries only `Stage` plus `Event`, the transform
is not preserving a source field here; it is manufacturing one.

## Trigger

Export contract events for a Soroban transaction that carries a transaction-level
event outside contract-call execution, such as a transaction-level system or
fee-related event. The resulting row will report
`in_successful_contract_call=true`, and in failed-transaction cases can even
produce the contradictory pair `successful=false` plus
`in_successful_contract_call=true`.

## Target Code

- `internal/transform/contract_events.go:30-38` — transaction-level events are routed through the synthetic conversion path
- `internal/transform/contract_events.go:142-148` — `transactionEvent2DiagnosticEvent()` hard codes `InSuccessfulContractCall: true`
- `internal/transform/contract_events.go:194-195,224-238` — the fabricated bool is copied into `ContractEventOutput`
- `internal/transform/contract_events_test.go:58-75,94-110` — tests already accept rows with `Successful: false` and `InSuccessfulContractCall: true`

## Evidence

The conversion helper does not inspect the transaction result, event type, or
event stage before setting the flag to `true`. The package tests already encode
that behavior: the expected output includes rows where `successful` is `false`
but `in_successful_contract_call` is `true`, showing the transform is emitting a
boolean that did not come from the wire format.

## Anti-Evidence

If stellar-core only emits `TransactionEvents` for successful contract-call
paths in the common cases seen so far, the fabricated `true` may often appear
plausible. But `TransactionEvent` metadata does not contain this boolean, so the
current export cannot distinguish a genuinely successful-contract-call context
from a transaction-level system event.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full code path from `TransformContractEvent` through `transactionEvent2DiagnosticEvent` to `parseDiagnosticEvent` and the final `ContractEventOutput` struct. Confirmed that `xdr.TransactionEvent` carries only `Stage` (an enum: `BeforeAllTxs`, `AfterTx`, `AfterAllTxs`) and `Event` (a `ContractEvent`) — no `InSuccessfulContractCall` field exists. The conversion at line 146 fabricates `InSuccessfulContractCall: true` unconditionally. The XDR specification for `TransactionMetaV4.Events` explicitly documents these as "Used for transaction-level events (like fee payment)", confirming they are system-level events not inherently tied to successful contract calls. The upstream SDK's token_transfer processor processes `TransactionEvents` directly without fabricating this boolean, demonstrating that no upstream convention mandates the fabrication.

### Code Paths Examined

- `internal/transform/contract_events.go:21-68` — `TransformContractEvent` routes `transactionEvents.TransactionEvents` through `transactionEvent2DiagnosticEvent` at line 32, then into `parseDiagnosticEvent` at line 33
- `internal/transform/contract_events.go:142-148` — `transactionEvent2DiagnosticEvent` unconditionally sets `InSuccessfulContractCall: true` with misleading comment "TransactionEvents are only emitted in successful contract calls"
- `internal/transform/contract_events.go:194,230` — `parseDiagnosticEvent` reads the fabricated bool at line 194 and writes it to output at line 230
- `go-stellar-sdk/xdr/xdr_generated.go:18177-18180` — `TransactionEvent` struct: `{Stage TransactionEventStage, Event ContractEvent}` — no `InSuccessfulContractCall` field
- `go-stellar-sdk/xdr/xdr_generated.go:18247-18270` — `TransactionMetaV4` XDR comment: `TransactionEvent events<>; // Used for transaction-level events (like fee payment)`
- `go-stellar-sdk/xdr/xdr_generated.go:17253-17255` — `DiagnosticEvent` struct: `{InSuccessfulContractCall bool, Event ContractEvent}` — this is the type that actually carries the boolean
- `go-stellar-sdk/ingest/ledger_transaction.go:278-313` — `GetTransactionEvents` extracts `txMeta.Events` for V4 metadata; no transformation of the boolean occurs at the SDK level
- `go-stellar-sdk/processors/token_transfer/contract_events.go:43-70` — upstream SDK processes `TransactionEvents` directly via `ev.Event` without fabricating `InSuccessfulContractCall`
- `internal/transform/contract_events_test.go:58-65` — test expects `Successful: false, InSuccessfulContractCall: true`, codifying the contradictory state
- `internal/transform/contract_events.go:151-159` — `contractEvent2DiagnosticEvent` also hard-codes `true`, following the same pattern for operation-level events (though OperationEvents arguably have a stronger claim to being in-call)

### Findings

1. **Fabricated boolean with no wire-format source**: `TransactionEvent` does not carry `InSuccessfulContractCall`. The conversion manufactures this value. The XDR type has `Stage` (timing enum) and `Event` (the event payload) — nothing about contract-call success context.

2. **XDR documentation contradicts the code comment**: The XDR comment for `TransactionMetaV4.Events` says "Used for transaction-level events (like fee payment)." Fee payment events are system-level operations, not contract calls. The code comment at line 143 ("TransactionEvents are only emitted in successful contract calls") appears to be incorrect.

3. **Stage values indicate non-contract-call context**: `TransactionEventStage` has `BeforeAllTxs` (events before any transaction), `AfterTx` (events after a transaction), and `AfterAllTxs` (events after all transactions). `BeforeAllTxs` and `AfterAllTxs` are ledger-wide stages that cannot be "inside" any specific contract call.

4. **Contradictory export state**: Tests already encode `Successful: false` + `InSuccessfulContractCall: true`. A downstream consumer filtering for `in_successful_contract_call=true` would incorrectly include fee-processing events from failed transactions.

5. **Upstream SDK does not fabricate**: The `go-stellar-sdk` token_transfer processor processes `TransactionEvents` directly using `ev.Event` without wrapping them in `DiagnosticEvent` or setting `InSuccessfulContractCall`. There is no upstream precedent for this fabrication.

6. **Stage information is lost**: The conversion discards `TransactionEvent.Stage` (as noted in the code comment at line 140). This means the export loses the timing context that could help distinguish fee events from other event types.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go`
- **Setup**: Create an `ingest.LedgerTransaction` with `TransactionMetaV4` containing a `TransactionEvent` with `Stage: TransactionEventStageTransactionEventStageBeforeAllTxs` and a fee-related `ContractEvent`. Set the transaction result to failed (`Successful() == false`).
- **Steps**: Call `TransformContractEvent(transaction, lhe)` and inspect the resulting `ContractEventOutput` for the TransactionEvent-sourced row.
- **Assertion**: Assert that the output row has `Successful: false` AND `InSuccessfulContractCall: true` — demonstrating the contradictory fabricated state. Then verify that the source `xdr.TransactionEvent` struct has no `InSuccessfulContractCall` field (compile-time verification). This proves the exported boolean has no wire-format backing and is unconditionally fabricated.
