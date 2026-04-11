# H053: Parallel generalized tx-set ordering does not yet prove wrong ledger counts

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural count mismatch in `history_ledgers`
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For ledgers encoded with generalized transaction sets and parallel execution stages, `TransformLedger()` should pair each transaction envelope with its matching result when computing `transaction_count`, `operation_count`, `successful_transaction_count`, and `failed_transaction_count`. A ledger containing mixed success/failure results and heterogeneous operation counts should therefore produce the same counts regardless of whether its transactions were serialized in classic or parallel tx-set form.

## Mechanism

`extractCounts()` flattens `TransactionHistoryEntry.Ext.GeneralizedTxSet.V1TxSet.Phases` through `GetTransactionSet()` and then zips the resulting envelope slice against `TransactionResult.TxResultSet.Results` by index. If the generalized tx-set phase/stage order differed from the result-set order, success flags and operation counts could be attributed to the wrong transactions, silently skewing the ledger aggregates.

## Trigger

Construct or locate a generalized tx-set ledger with at least one parallel phase containing transactions that differ in both operation count and success/failure outcome. Compare the exported ledger counts against counts recomputed from `LedgerCloseMeta.TxProcessing`; a mismatch would prove the zip order is wrong.

## Target Code

- `internal/transform/ledger.go:133-165` — `extractCounts()` zips flattened envelopes against `TxResultSet.Results`
- `internal/transform/ledger.go:168-209` — `GetTransactionSet()` flattens generalized tx-set phases, stages, and clusters
- `internal/input/ledgers.go:33-77` — captive-core path reconstructs `historyarchive.Ledger` by appending `TxProcessing` results in processing order
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/ledger_close_meta.go:62-93` — upstream helper flattens transaction envelopes with the same phase/stage walk

## Evidence

The transform code has to reconcile two differently shaped structures: a phased generalized tx set on one side and a flat result slice on the other. The XDR comments for `ParallelTxsComponent` note that stage execution order matters, so it is reasonable to question whether naive flattening preserves the same indexing contract as `TransactionResultSet.Results`.

## Anti-Evidence

The concrete code paths I traced all flatten envelopes and results in what appears to be the same processing order. `internal/input/ledgers.go` rebuilds `TransactionResult.TxResultSet.Results` directly from `LedgerCloseMeta.TxProcessing`, and the upstream `xdr.LedgerCloseMeta.TransactionEnvelopes()` helper uses the same nested phase/stage/cluster walk as `GetTransactionSet()`. I did not find a concrete ledger format contract or test fixture demonstrating that generalized tx-set envelope order diverges from result order.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not establish a concrete desynchronization between the flattened generalized tx-set order and the result-set order. The local captive-core reconstruction path and the upstream `LedgerCloseMeta` helper both use matching processing-order traversals, so this remains suspicion without a reproducible mismatch.

### Lesson Learned

Parallel-stage metadata alone is not enough to claim count corruption. For generalized tx-set hypotheses, I need a concrete example or format contract showing that transaction envelopes and result pairs are flattened in different orders before treating the index zip as a bug.
