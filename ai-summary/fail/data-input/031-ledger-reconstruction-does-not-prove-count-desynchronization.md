# H031: Reconstructed `historyarchive.Ledger` does not prove envelope/result count desynchronization

**Date**: 2026-04-15
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `GetLedgers()` reconstructs a `historyarchive.Ledger` from `LedgerCloseMeta`, the transaction envelopes and transaction results should stay in the same processing order so `TransformLedger()` computes accurate `transaction_count`, `successful_transaction_count`, and `failed_transaction_count`.

## Mechanism

`GetLedgers()` builds the reconstructed ledger from two different sources: it fills `Transaction.TxSet.Txs` from `lcm.TransactionEnvelopes()` and fills `TransactionResult.TxResultSet.Results` by iterating `lcm.V{0,1,2}.TxProcessing`. If those traversals used different ordering rules, `extractCounts()` in the ledger transform could attribute success and failure results to the wrong envelopes and silently publish incorrect aggregate counts.

## Trigger

Run `export_ledgers` on a ledger whose `LedgerCloseMeta` contains multiple transactions, especially protocol versions with generalized transaction sets, and compare the exported success/failure counts against the authoritative processing results. The suspected trigger is any ledger where tx-set ordering diverges from transaction-processing ordering.

## Target Code

- `internal/input/ledgers.go:GetLedgers:13-90` — reconstructs `historyarchive.Ledger` from `TransactionEnvelopes()` plus `TxProcessing`
- `internal/transform/ledger.go:TransformLedger:16-36` — derives ledger counts from the reconstructed `historyarchive.Ledger`

## Evidence

The reconstruction path really does zip together envelope data from `TransactionEnvelopes()` and result data from `TxProcessing`, rather than preserving a single pre-paired upstream object. That makes ordering assumptions a legitimate thing to audit.

## Anti-Evidence

After tracing the local reconstruction and adjacent prior work, there is still no concrete demonstration that the SDK helpers return envelopes in an order different from the transaction-processing results consumed here. The published and failed investigations around generalized tx-set ordering already point in the opposite direction: both the local reconstruction path and the upstream helper logic appear to follow processing order consistently.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The code shape makes an ordering bug plausible, but I could not establish an actual order mismatch. Without a concrete fixture, protocol rule, or SDK contract showing that `TransactionEnvelopes()` and `TxProcessing` diverge, this remains an unproven suspicion rather than a viable data-integrity finding.

### Lesson Learned

In this subsystem, "two arrays are zipped together" is only a starting point. Before treating it as corruption, verify the upstream ordering contract or find a real ledger format where the two traversals differ; otherwise the hypothesis collapses into a structural resemblance to a bug rather than a demonstrated output defect.
