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

The current helper comment claims transaction events are only emitted in successful contract calls, but upstream XDR describes transaction events as transaction-level events such as fee payment/refund. The repository's own test data demonstrates the contradiction: it exports a row with `Successful: false` while keeping `InSuccessfulContractCall: true`.

## Anti-Evidence

Operation-scoped `ContractEvent` rows in `OperationEvents` are expected to come from successful contract execution, so the hard-coded `true` is not obviously wrong for that stream. The corruption is specific to transaction-level events that are not themselves contract-call events.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full data path from `TransformContractEvent()` through `transactionEvent2DiagnosticEvent()` and `parseDiagnosticEvent()`. The XDR definition of `TransactionMetaV4.Events` explicitly states these are "Used for transaction-level events (like fee payment)" — not contract-call events. The `TransactionEventStage` enum has stages like `BEFORE_ALL_TXS` ("before any one of the transactions has its operations applied"), which by definition cannot occur inside a contract call. The hard-coded `InSuccessfulContractCall: true` in `transactionEvent2DiagnosticEvent()` is based on a factually incorrect comment and produces semantically wrong output for all transaction-level events.

### Code Paths Examined

- `internal/transform/contract_events.go:transactionEvent2DiagnosticEvent:139-149` — Hard-codes `InSuccessfulContractCall: true` with comment "TransactionEvents are only emitted in successful contract calls" which contradicts the XDR specification
- `internal/transform/contract_events.go:TransformContractEvent:31-38` — Iterates `transactionEvents.TransactionEvents` and passes each through the lossy conversion
- `internal/transform/contract_events.go:parseDiagnosticEvent:167-242` — Copies `diagnosticEvent.InSuccessfulContractCall` (the fabricated `true`) into `ContractEventOutput.InSuccessfulContractCall` at line 194/230
- `internal/transform/contract_events.go:contractEvent2DiagnosticEvent:152-159` — Same pattern for operation-level `ContractEvent`, but this one is defensible since operation events ARE from successful execution
- `internal/transform/contract_events_test.go:93-110` — Test expects `Successful: false` + `InSuccessfulContractCall: true` for a `TransactionEvent` with `Stage: BEFORE_ALL_TXS` on a `TxFailed` transaction — confirming the contradiction in the test fixture
- `internal/transform/schema.go:ContractEventOutput:641-657` — Schema exposes `in_successful_contract_call` as a `bool` JSON field consumed by BigQuery
- XDR `TransactionMetaV4` struct comment: `TransactionEvent events<>; // Used for transaction-level events (like fee payment)` — confirms these are NOT contract-call events
- XDR `TransactionEventStage` enum: three stages (`BEFORE_ALL_TXS=0`, `AFTER_TX=1`, `AFTER_ALL_TXS=2`) describe transaction/ledger lifecycle phases, not contract invocation context

### Findings

1. **Factually incorrect code comment**: The comment at line 143 states "TransactionEvents are only emitted in successful contract calls." The XDR definition of `TransactionMetaV4.Events` says the opposite: these are "Used for transaction-level events (like fee payment)." The `TransactionEventStage` enum describes ledger/transaction lifecycle phases (before all txs, after tx, after all txs), none of which are inside a contract call.

2. **Hard-coded wrong value**: `transactionEvent2DiagnosticEvent()` unconditionally sets `InSuccessfulContractCall: true` for every `TransactionEvent`. Since `TransactionEvent` represents transaction-level lifecycle events (fee payment, fee refund), not contract invocations, this value is semantically wrong for all events in this category.

3. **Test confirms the contradiction**: The test fixture at lines 94-109 expects `Successful: false` combined with `InSuccessfulContractCall: true` for a `TransactionEvent` at stage `BEFORE_ALL_TXS` on a `TxFailed` transaction. An event that happened "before any transaction operations were applied" on a failed transaction cannot have occurred "in a successful contract call."

4. **Distinct from the Stage-dropping issue (H003/fail 011)**: The previously investigated hypothesis about dropping the `Stage` field was rejected as working-as-designed. This hypothesis addresses a different problem: the fabricated `InSuccessfulContractCall` boolean value. Dropping `Stage` was an explicit design choice with a documenting comment. Setting `InSuccessfulContractCall: true` is based on a factually wrong assumption in the comment.

5. **Downstream impact**: BigQuery consumers querying `WHERE in_successful_contract_call = true` will include transaction-level fee events that never occurred in a contract call. Queries filtering on `WHERE successful = false AND in_successful_contract_call = true` would return contradictory rows.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go`
- **Setup**: Create a `TransactionMetaV4` with a `TransactionEvent` at stage `BEFORE_ALL_TXS` on a failed transaction (`TransactionResultCodeTxFailed`). The existing test helper `makeContractEventTestInput()` already does this for the second test case (index 1, lines 210-308 and 389-442).
- **Steps**: Call `TransformContractEvent()` with the test transaction and inspect the output for events sourced from `transactionEvents.TransactionEvents`.
- **Assertion**: Assert that for the `TransactionEvent`-sourced output row, `InSuccessfulContractCall` is `false` (not `true`). Currently, the test at lines 94-109 asserts the opposite (`true`), confirming the bug. A corrected implementation should set `InSuccessfulContractCall: false` for all `TransactionEvent` entries, since they represent transaction-level lifecycle events, not contract-call events.
