# H002: V4 Soroban diagnostic events still export `operation_id = NULL` despite a single parent operation

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a `TransactionMetaV4` Soroban transaction, exported diagnostic events should retain the TOID of the single `InvokeHostFunction` operation that owns the transaction's contract execution. Diagnostic rows from that transaction should therefore join to the same `history_operations.id` as the corresponding contract-event rows.

## Mechanism

Post-CAP-67 ingest still preserves the fact that Soroban transactions only expose one contract-bearing operation, but `TransformContractEvent()` keeps treating `DiagnosticEvents` as TOID-less rows. That makes V4 diagnostic rows look transaction-scoped even when the surrounding transaction has exactly one operation and the sibling `OperationEvents` rows already prove the correct operation TOID.

## Trigger

Run `export_contract_events` on a Soroban ledger with unified events (`TransactionMeta.V == 4`) and at least one diagnostic event, then compare the diagnostic rows' null `operation_id` against the sole `history_operations.id` for the transaction.

## Target Code

- `internal/transform/contract_events.go:42-65` — op TOIDs are populated for `OperationEvents`, but the later `DiagnosticEvents` loop appends rows without `OperationID`.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:300-307` — V4 `GetTransactionEvents()` preserves per-operation events by slot while exposing diagnostics as a flat array.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction_test.go:1052-1065` — upstream tests document that V4 Soroban transactions still "only ever have 1 operation".
- `internal/transform/contract_events_test.go:128-144` — current V4 expected output leaves the diagnostic-event row's `OperationID` empty even though the sibling operation-event row uses op TOID `...729`.

## Evidence

The local transform already derives the right operation TOID for V4 operation events from the same transaction, so the null diagnostic row is not caused by missing ledger or transaction context. The upstream ingest/test contract that Soroban V4 transactions still have one operation makes the parent operation deterministic, not ambiguous.

## Anti-Evidence

`DiagnosticEvents` are not nested under `OperationMetaV2`, so the current model may intentionally treat them as transaction-scoped metadata even in one-operation Soroban transactions. A reviewer should confirm whether this null is a deliberate semantic choice or an avoidable loss of a stable join key.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of fail/toid/012-v3-diagnostic-events-miss-operation-toid
**Failed At**: reviewer

### Trace Summary

This hypothesis targets V4 diagnostic events but is substantively identical to the already-investigated fail/toid/012, which covered V3 diagnostic events and explicitly analyzed V4 as part of its cross-version consistency check. The review of 012 concluded: "V4 also treats `txMeta.DiagnosticEvents` as transaction-scoped, confirming cross-version consistency." The same code path (`contract_events.go:58-64`) handles diagnostic events identically regardless of meta version.

### Code Paths Examined

- `internal/transform/contract_events.go:58-64` — DiagnosticEvents loop does not assign OperationID; identical path for V3 and V4
- `internal/transform/contract_events_test.go:128-144` — V4 test explicitly expects `OperationID: null.Int{}` for diagnostic events
- `internal/transform/contract_events_test.go:109` — V3 test also expects `OperationID: null.Int{}`, confirming cross-version consistency

### Why It Failed

This is a duplicate of fail/toid/012. That investigation already covered the V4 case explicitly, finding that:

1. **SDK design**: `GetTransactionEvents()` separates contract events (operation-scoped, in `OperationEvents`) from diagnostic events (transaction-scoped, in `DiagnosticEvents`) for both V3 and V4.
2. **Cross-version consistency**: Both V3 and V4 treat diagnostic events identically as transaction-scoped. The 012 review explicitly confirmed this.
3. **Intentional test expectations**: Tests at `contract_events_test.go:109` (V3) and `:143` (V4) both expect null `OperationID` for diagnostic events.

The hypothesis simply re-states the V3 claim from 012 but scoped to V4, which was already analyzed and rejected as working-as-designed behavior.

### Lesson Learned

When a prior investigation explicitly covers multiple versions (V3 and V4) in its cross-version consistency analysis, a new hypothesis targeting a different version of the same behavior is a duplicate.
