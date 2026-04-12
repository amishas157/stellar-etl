# H067: Transaction-wide diagnostic scan cannot cross-contaminate `asset_balance_changes` across operations

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

An `invoke_contract` operation's `asset_balance_changes` should reflect only the
SAC balance deltas caused by that operation, not deltas from another operation
in the same transaction.

## Mechanism

`extractOperationDetails()` looked suspicious because the live invoke-contract
path calls `parseAssetBalanceChangesFromContractEvents(transaction, network)`,
which scans `transaction.GetDiagnosticEvents()` at transaction scope instead of
reading an operation-scoped event slice. If a Soroban transaction could contain
multiple contract-executing operations, one operation row might therefore pick
up another operation's SAC balance changes.

## Trigger

Export `history_operations` for a transaction containing multiple
contract-executing operations with disjoint SAC transfers, then compare each
row's `asset_balance_changes` against its own operation-scoped events.

## Target Code

- `internal/transform/operation.go:1093-1095` — invoke-contract details call the transaction-wide helper
- `internal/transform/operation.go:1942-1974` — helper scans `transaction.GetDiagnosticEvents()` for SAC balance changes
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:249-257` — SDK `GetContractEvents()` documents the single-smart-contract-operation assumption
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:268-299` — transaction-event breakdown shows diagnostics are transaction-scoped

## Evidence

The helper is undeniably transaction-wide, and the operation detail row it feeds
is operation-scoped. Without a protocol invariant, that would be a plausible
cross-operation attribution bug.

## Anti-Evidence

The SDK explicitly documents that Soroban smart-contract transactions rely on
there being only one smart-contract operation in the transaction, and the
diagnostic events are transaction-scoped by design. That invariant makes the
cross-operation contamination trigger unreachable for legitimate chain data.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected corruption requires multiple contract-executing operations sharing
one diagnostic-event bucket, but the upstream Soroban transaction model used by
this path assumes a single smart-contract operation per transaction. With only
one relevant operation, the transaction-wide diagnostic scan cannot misattribute
another operation's SAC balance changes onto the current row.

### Lesson Learned

A transaction-scoped helper is only a live attribution bug if the protocol
actually permits multiple relevant producers inside that transaction. For
Soroban event plumbing, verify the upstream operation-count invariant before
concluding that transaction-wide diagnostics can leak across operation rows.
