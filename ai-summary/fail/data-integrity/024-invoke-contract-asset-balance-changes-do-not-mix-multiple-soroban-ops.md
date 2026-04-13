# H024: `invoke_contract` asset-balance changes do not mix multiple Soroban operations

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`details.asset_balance_changes` for an `invoke_contract` operation should include only the Stellar Asset Contract events emitted by that operation. A transaction containing multiple Soroban operations should not let one operation's balance-change list absorb another operation's diagnostic events.

## Mechanism

`parseAssetBalanceChangesFromContractEvents()` reads `transaction.GetDiagnosticEvents()` at transaction scope and does not filter by operation index. If a single Soroban transaction could legitimately contain multiple Soroban operations, the helper would append sibling operations' SAC events into the current operation's exported `asset_balance_changes`, producing plausible but wrong balance-change rows.

## Trigger

Construct a transaction with two Soroban operations that both emit SAC diagnostic events, then export operation details for the first operation.

## Target Code

- `internal/transform/operation.go:parseAssetBalanceChangesFromContractEvents:1907-1939` — reads transaction-wide diagnostic events with no operation filter
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/ingest/ledger_transaction.go:GetDiagnosticEvents/GetTransactionEvents:252-298` — upstream ingest comments explain the single-operation Soroban invariant and build one-slot `OperationEvents` for Soroban txmeta

## Evidence

The local helper really does operate on transaction-wide diagnostic events, not per-operation event slices. If multiple Soroban operations were legal in one transaction, the current code would have no way to keep their SAC balance changes separated.

## Anti-Evidence

The trigger is impossible in the currently supported protocol surface. Upstream ingest explicitly relies on the fact that Soroban transactions contain only one operation, and its Soroban event reconstruction builds a single operation slot on that basis. That invariant prevents the transaction-wide diagnostic scan from cross-contaminating multiple Soroban operations today.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected cross-operation contamination depends on a multi-Soroban-operation transaction shape that the current protocol and upstream ingest model do not allow. With only one Soroban operation per transaction, the transaction-wide diagnostic slice and the operation-local event set collapse to the same event population.

### Lesson Learned

Transaction-scoped event helpers are only a live integrity risk when the protocol permits multiple independently eventful operations in one transaction. Before escalating "wrong scope" hypotheses on Soroban metadata, verify the current one-operation invariant in the upstream ingest/XDR layer.
