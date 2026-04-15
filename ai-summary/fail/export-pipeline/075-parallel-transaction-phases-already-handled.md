# H075: Ledger transaction-phase flattening already supports parallel phases

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `history_ledgers` is built from a generalized transaction set containing `TransactionPhase.V == 1`, the local phase flattener should traverse `ParallelTxsComponent.ExecutionStages` rather than panicking on an unknown phase type.

## Mechanism

An initial partial read of `getTransactionPhase()` suggested it only handled `V0Components`, which would have made `export_ledgers` crash or undercount transactions as soon as a ledger used the newer parallel phase arm. That would have been a clean live divergence from the SDK's own `LedgerCloseMeta.TransactionEnvelopes()` helper.

## Trigger

Export a ledger whose generalized transaction set contains a `TransactionPhase` with `V == 1` and compare the local flattener against the current SDK helper.

## Target Code

- `internal/transform/ledger.go:getTransactionPhase:181-209` — local generalized-tx-set flattener
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/ledger_close_meta.go:62-93` — SDK helper flattens both classic and parallel phases

## Evidence

The repo carries its own transaction-phase flattening helper instead of delegating to `LedgerCloseMeta.TransactionEnvelopes()`, so stale phase handling is a realistic failure mode whenever the protocol adds a new arm.

## Anti-Evidence

The full local function already includes `case 1:` and iterates `phase.ParallelTxsComponent.ExecutionStages` in the same shape as the SDK helper. The initial suspicion came from an incomplete file slice, not from the actual implementation.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`getTransactionPhase()` already handles `TransactionPhase.V == 1`. There is no missing parallel-phase support on the current code path.

### Lesson Learned

Custom flatteners are worth auditing, but only after reading the complete function body. Partial-context scans are especially dangerous in large switch statements where the relevant arm may sit just below the initial view range.
