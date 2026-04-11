# H005: Transaction-level fee events are mislabeled as successful contract calls

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_contract_events` should preserve the true `in_successful_contract_call` semantics of exported rows. Transaction-level fee/refund events, especially on failed Soroban transactions, should not be labeled as if they happened inside a successful contract call.

## Mechanism

Every `TransactionEvent` is rewritten into a synthetic `DiagnosticEvent` with `InSuccessfulContractCall: true`, regardless of transaction outcome or event stage. `parseDiagnosticEvent()` then copies that hard-coded boolean straight into `ContractEventOutput`, so failed-transaction fee events export with a contradictory combination like `successful=false` and `in_successful_contract_call=true`.

## Trigger

Run `export_contract_events` on a failed Soroban transaction that still emits transaction-level fee/refund events in `TransactionMetaV4.Events`. The exported row will claim `in_successful_contract_call=true` even though the enclosing transaction failed and the event did not come from a successful contract invocation.

## Target Code

- `internal/transform/contract_events.go:transactionEvent2DiagnosticEvent:139-149` — hard-codes `InSuccessfulContractCall: true` for every `TransactionEvent`
- `internal/transform/contract_events.go:TransformContractEvent:31-38` — applies that conversion to all transaction-level events
- `internal/transform/contract_events.go:parseDiagnosticEvent:194-239` — persists the fabricated boolean to output
- `internal/transform/contract_events_test.go:94-109` — repository fixture already expects a failed transaction row with `InSuccessfulContractCall: true`

## Evidence

The current helper comment claims transaction events are only emitted in successful contract calls, but upstream XDR describes transaction events as transaction-level events such as fee payment/refund. The repository’s own test data demonstrates the contradiction: it exports a row with `Successful: false` while keeping `InSuccessfulContractCall: true`.

## Anti-Evidence

Operation-scoped `ContractEvent` rows in `OperationEvents` are expected to come from successful contract execution, so the hard-coded `true` is not obviously wrong for that stream. The corruption is specific to transaction-level events that are not themselves contract-call events.
