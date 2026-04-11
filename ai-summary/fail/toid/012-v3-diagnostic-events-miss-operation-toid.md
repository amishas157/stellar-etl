# H001: V3 Soroban diagnostic events lose a derivable operation TOID

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a `TransactionMetaV3` Soroban transaction, every exported diagnostic-event row should carry the TOID of the single `InvokeHostFunction` operation that produced it. Downstream joins from `history_contract_events.operation_id` to `history_operations.id` should therefore work for both contract events and diagnostic events from the same execution.

## Mechanism

The V3 ingest path already collapses Soroban contract events into exactly one operation slot, which means the parent operation is unambiguous. But `TransformContractEvent()` only assigns `OperationID` inside the `OperationEvents` loop and appends every V3 diagnostic event from `transactionEvents.DiagnosticEvents` with a null `operation_id`, silently discarding a join key that can still be derived from `(ledger sequence, transaction index, op=1)`.

## Trigger

Run `export_contract_events` on a pre-unified-events Soroban ledger (`TransactionMeta.V == 3`) that contains diagnostic events, then compare the exported diagnostic rows against the sole `history_operations.id` for the parent `InvokeHostFunction` transaction.

## Target Code

- `internal/transform/contract_events.go:41-65` — only the `OperationEvents` branch assigns `OperationID`; the `DiagnosticEvents` branch does not.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:278-299` — V3 `GetTransactionEvents()` synthesizes exactly one operation-event slot for Soroban transactions and returns diagnostics separately.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:252-258` — `GetContractEvents()` explicitly relies on there being only one Soroban operation in the transaction.
- `internal/transform/contract_events_test.go:58-91` — current expected output leaves the diagnostic-event row's `OperationID` empty even when the sibling contract-event row from the same transaction uses op TOID `...633`.

## Evidence

The transform already computes the canonical operation TOID for the matching V3 contract-event rows, so all ingredients needed to populate the diagnostic row are present in-process. Because V3 Soroban transactions are modeled as single-operation transactions by the ingest layer, leaving the diagnostic row null is not preserving ambiguity; it is dropping a deterministic parent ID.

## Anti-Evidence

The XDR `DiagnosticEvent` itself does not carry an explicit operation index, and the current tests intentionally treat diagnostics as a separate stream. A reviewer should confirm that consumers expect V3 diagnostics to join back to `history_operations` rather than remain transaction-scoped by convention.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not an exact duplicate, but follows the same reasoning pattern as fail/toid/011 (token-transfer fee events)
**Failed At**: reviewer

### Trace Summary

Traced the complete V3 diagnostic event path from the SDK's `GetTransactionEvents()` through `TransformContractEvent()`. The SDK deliberately separates V3 Soroban events into two buckets: contract events go into `OperationEvents[0]` (operation-scoped), while diagnostic events go into `DiagnosticEvents` (transaction-scoped). The ETL faithfully preserves this upstream scope distinction. The behavior is consistent between V3 and V4 meta versions — in both cases, diagnostic events are transaction-scoped and lack operation IDs.

### Code Paths Examined

- `go-stellar-sdk/ingest/ledger_transaction.go:GetTransactionEvents():V3 case` — SDK explicitly puts contract events into `OperationEvents[0]` but diagnostic events into a separate `DiagnosticEvents` bucket, establishing them as transaction-scoped
- `internal/transform/contract_events.go:42-55` — OperationEvents loop assigns OperationID via TOID for operation-scoped events
- `internal/transform/contract_events.go:58-64` — DiagnosticEvents loop correctly preserves the transaction-scoped nature by not assigning OperationID
- `internal/transform/contract_events_test.go:109,143` — Test expected output explicitly sets `OperationID: null.Int{}` for diagnostic events in both V3 and V4 cases, confirming this is intentional
- `go-stellar-sdk/ingest/ledger_transaction.go:GetTransactionEvents():V4 case` — V4 also treats `txMeta.DiagnosticEvents` as transaction-scoped, confirming cross-version consistency

### Why It Failed

The hypothesis describes **working-as-designed behavior**. Three independent lines of evidence confirm this:

1. **SDK design**: `GetTransactionEvents()` deliberately separates contract events (operation-scoped, placed in `OperationEvents`) from diagnostic events (transaction-scoped, placed in `DiagnosticEvents`). The XDR `DiagnosticEvent` type itself carries no operation index — the scope is an inherent property of the event category.

2. **Cross-version consistency**: Both V3 and V4 treat diagnostic events identically as transaction-scoped. If V3 diagnostics were meant to be operation-scoped, V4 would need to handle them differently (since V4 supports multi-operation transactions), but it doesn't — they remain uniformly transaction-scoped.

3. **Intentional test expectations**: The test at `contract_events_test.go:109` and `:143` explicitly expects `OperationID: null.Int{}` for diagnostic event rows in both V3 and V4 test cases.

The hypothesis argues that because V3 Soroban transactions have exactly one operation, the diagnostic events "should" inherit that operation's TOID. But this conflates "derivable" with "correct." The upstream event model explicitly treats diagnostics as transaction-scoped, and inventing an operation TOID locally would misrepresent the event's actual scope.

### Lesson Learned

Do not "upgrade" a null operation TOID just because the surrounding transaction has one operation. The SDK deliberately separates event categories by scope (operation vs transaction). Diagnostic events are transaction-scoped by design across all meta versions. This is the same principle documented in fail/toid/011 for token-transfer fee events.
