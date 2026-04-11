# H013: P23 fee-refund accounting does not double-count `PostTxApplyFeeChanges`

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Protocol 23+ Soroban transactions, `resource_fee_refund` should include the fee-account refund balance change exactly once. If the same refund delta were present in both `TxChangesAfter` and `PostTxApplyFeeChanges`, appending the two slices would double-count the refund and export an inflated value.

## Mechanism

`TransformTransaction()` builds `feeChanges := append(metav4.TxChangesAfter, transaction.PostTxApplyFeeChanges...)` before passing the combined slice into `getAccountBalanceFromLedgerEntryChanges()`. That initially looked like a version-skew bug: if the SDK already duplicated the refund delta into both collections, the exporter would silently overstate `resource_fee_refund`.

## Trigger

Process a Protocol 23+ Soroban transaction where the same fee-account refund balance change appears in both metadata locations.

## Target Code

- `internal/transform/transaction.go:195-203` — appends `TxChangesAfter` and `PostTxApplyFeeChanges`
- `github.com/stellar/go-stellar-sdk/ingest/ledger_transaction.go:21-27, 60-69` — documents `PostTxApplyFeeChanges`
- `github.com/stellar/go-stellar-sdk/ingest/ledger_transaction.go:115-164` — upstream change reader contract for v4 transaction meta

## Evidence

At the repository level, the export code literally concatenates both change lists before computing the refund. Without an upstream guarantee that the two lists are disjoint, this would be a real financial double-count.

## Anti-Evidence

The current upstream ingest package explicitly documents the opposite contract: for P23 onwards, the refund balance changes move to `PostTxApplyFeeChanges` and do not remain in `TxChangesAfter`, and it states that callers can safely append the two slices before scanning them. The reader implementation also sources `PostTxApplyFeeChanges` from a distinct `LedgerCloseMetaV2.TxProcessing[i].PostTxApplyFeeProcessing` field.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected double-count is ruled out by the current upstream ingest contract. The ETL code is following the SDK's documented P23+ fee-refund guidance rather than compensating for an already-duplicated delta.

### Lesson Learned

When a transform composes low-level ingest fields, check the SDK's own invariants before treating the composition as suspicious. Here, the authoritative upstream docs convert an apparent double-count into the intended compatibility pattern.
