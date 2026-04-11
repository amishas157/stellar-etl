# H028: Contract-event `operation_id` does not drift when earlier operations emit no events

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When only a later operation in a transaction emits Soroban contract events, the exported `operation_id` should still point at that real parent operation. Earlier operations with zero events must not cause the ETL to renumber the later event rows.

## Mechanism

`TransformContractEvent()` derives `operation_id` from the outer-slice index `i` of `transactionEvents.OperationEvents`, so it initially looks vulnerable if the upstream SDK compacts away empty per-operation event lists. In that failure mode, an event from operation 3 could be exported as operation 1.

## Trigger

Use a transaction with multiple operations where only a later operation emits contract events, then export contract events and compare the emitted `operation_id` against the parent operation TOID.

## Target Code

- `internal/transform/contract_events.go:41-52` — derives `operation_id` from the `OperationEvents` slice index.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:300-306` — upstream `GetTransactionEvents()` allocates `OperationEvents` to `len(txMeta.Operations)` and copies each operation's `Events` slice into the matching slot.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:296-299` — V3 Soroban transactions only ever expose a single smart-contract operation slot.

## Evidence

The local transform really does trust the slice position rather than reading an explicit operation index from each event. If the SDK had compacted the collection, the ETL would have exported wrong TOIDs with no local guard.

## Anti-Evidence

The upstream ingest implementation preserves empty slots explicitly for V4 metadata by allocating one `OperationEvents` entry per operation and copying `op.Events` by index. For V3 metadata, the upstream code documents that there is only one smart-contract operation, so the single-slot assumption also holds there.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The feared compaction never happens: upstream `GetTransactionEvents()` preserves per-operation slot alignment, so `i+1` in `TransformContractEvent()` remains the true operation index even when earlier operations emit zero events.

### Lesson Learned

Index-derived IDs are only suspect when the producer compacts collections. Before filing an off-by-N event-index bug, inspect the upstream container-construction code and verify whether empty slots are retained.
