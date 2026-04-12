# H003: Contract-event rows use parent transaction success instead of event-level call success

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: event-level success filtering can return wrong contract-event rows
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Because each `history_contract_events` row represents an individual contract or diagnostic event, the row-level `successful` flag should reflect whether that specific event came from a successful contract call. For diagnostic rows emitted with `InSuccessfulContractCall = true`, the exported row should likewise indicate success at the event level rather than inheriting the parent transaction's status wholesale.

## Mechanism

`parseDiagnosticEvent()` sets `Successful` from `transaction.Result.Successful()` and stores the event-local flag separately in `InSuccessfulContractCall`. That means the row-level `successful` column is transaction-scoped, not event-scoped, even though the table row itself is event-scoped. The checked-in test already demonstrates a split state (`Successful: false` with `InSuccessfulContractCall: true`), which makes naive filters on `successful` silently drop events that the companion event-local flag says came from a successful contract call.

## Trigger

1. Export contract-event rows for a transaction whose diagnostic event has `InSuccessfulContractCall = true` while the parent transaction-level success flag differs.
2. Filter rows using `successful = true`.
3. The ETL will omit event rows that were emitted from successful contract-call context because `successful` mirrors the parent transaction instead of the event itself.

## Target Code

- `internal/transform/contract_events.go:186-194` — parent transaction success and event-local success are read from different sources
- `internal/transform/contract_events.go:224-230` — row exports both values, but `successful` uses the transaction-level flag
- `internal/transform/contract_events_test.go:58-65` — current test fixture expects `Successful: false` and `InSuccessfulContractCall: true` on the same row

## Evidence

The test fixture proves the transform can emit rows whose `successful` flag disagrees with `InSuccessfulContractCall`, so this is not just a hypothetical naming concern. Since `successful` is a top-level column and many downstream filters will naturally use it first, the row shape is primed for event-level misclassification.

## Anti-Evidence

The schema may intentionally define `successful` as parent transaction success, with `InSuccessfulContractCall` as the more specific event-local discriminator. The existing test expectation suggests the current implementation is deliberate, even if it is easy for downstream consumers to misread.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `parseDiagnosticEvent()` in `contract_events.go` and confirmed that `Successful` (line 186) is sourced from `transaction.Result.Successful()` while `InSuccessfulContractCall` (line 194) is sourced from `diagnosticEvent.InSuccessfulContractCall`. The schema at `schema.go:641-657` defines both as separate fields with distinct JSON tags (`successful` and `in_successful_contract_call`). The same `transaction.Result.Successful()` pattern is used for the `Successful` field in `TransactionOutput` (transaction.go:232), confirming `successful` consistently means transaction-level success across the entire ETL.

### Code Paths Examined

- `internal/transform/contract_events.go:186` — `outputSuccessful := transaction.Result.Successful()` sets transaction-level success
- `internal/transform/contract_events.go:194` — `outputInSuccessfulContractCall := diagnosticEvent.InSuccessfulContractCall` sets event-level success
- `internal/transform/contract_events.go:224-239` — both fields are exported as separate columns in `ContractEventOutput`
- `internal/transform/schema.go:641-657` — `ContractEventOutput` struct defines `Successful bool` and `InSuccessfulContractCall bool` as distinct fields
- `internal/transform/transaction.go:232` — `TransactionOutput` uses the same `transaction.Result.Successful()` for its `Successful` field, confirming cross-entity consistency
- `internal/transform/contract_events_test.go:58-65` — test fixture explicitly expects `Successful: false` with `InSuccessfulContractCall: true`, confirming this is intentional design

### Why It Failed

This is **working-as-designed behavior**, not a bug. The `successful` field intentionally represents transaction-level success, consistent with the identical field name and semantics on `TransactionOutput`. The `in_successful_contract_call` field provides the separate event-level discriminator. Both fields serve distinct, well-defined semantic purposes:

1. `successful` answers: "Did the parent transaction succeed?" — useful for filtering out events from failed transactions entirely.
2. `in_successful_contract_call` answers: "Was this specific event emitted during a successful contract call?" — useful for event-level filtering within diagnostic events.

The schema deliberately provides both columns so downstream consumers can choose the appropriate granularity. A downstream consumer who wants event-level success filtering should use `in_successful_contract_call`, not `successful`. This is not a case of the ETL producing wrong data — it produces two correct, complementary signals.

### Lesson Learned

When a schema exports two related boolean columns with different scopes (transaction-level vs. event-level), verify whether the separation is intentional design before treating it as a bug. The presence of both fields with distinct names and the test fixture explicitly expecting the split state are strong signals of deliberate design.
