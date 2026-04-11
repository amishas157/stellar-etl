# H003: Multi-invoke operation details smear transaction-wide diagnostic balance changes

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

In a Soroban transaction with multiple `InvokeHostFunction` operations, each operation's `history_operations.details.asset_balance_changes` should describe only that operation's own SAC balance changes. If operation 1 transfers asset A and operation 2 mints asset B, the two exported operation rows should have different `asset_balance_changes` arrays that match their respective per-operation events.

## Mechanism

`parseAssetBalanceChangesFromContractEvents()` accepts only the whole transaction and network, not the current operation index. It reads the transaction-wide diagnostic stream, filters any embedded contract events, and returns the full set as a single array. The SDK warns that diagnostic streams may include contract events depending on txmeta generation. When that mirroring is enabled and a transaction contains multiple invoke operations, every invoke operation row receives the union of all mirrored SAC balance-change events in the transaction rather than just its own subset.

## Trigger

Export operations for a multi-`InvokeHostFunction` Soroban transaction generated with txmeta that mirrors contract events into `DiagnosticEvents`, where two invoke operations emit different SAC transfer/mint/burn/clawback events. Both exported operation rows will contain the same combined `asset_balance_changes` array even though each operation emitted only one of those changes.

## Target Code

- `internal/transform/operation.go:1093-1097` — invoke-host-function details always use the helper's returned array
- `internal/transform/operation.go:1942-1975` — helper reads transaction-wide diagnostic events and has no operation-index filter
- `internal/transform/operation.go:1886-1894` — `filterEvents()` converts all qualifying diagnostics into one contract-event slice
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/ingest/ledger_transaction.go:260-275` — SDK warns diagnostics may include contract events and describes V4's per-operation event model

## Evidence

The helper signature itself proves it cannot partition balance changes by operation. The only filtering applied is `InSuccessfulContractCall` plus `ContractEventTypeContract`, so once mirrored contract events from multiple invoke operations are present in diagnostics, the returned array becomes transaction-wide. This is distinct from the empty-array bug: one txmeta configuration drops the data entirely, while another configuration over-associates it to every invoke operation.

## Anti-Evidence

Many environments appear to generate diagnostics without mirrored contract events, in which case this path stays latent and hypothesis H001 manifests instead as an empty array. Single-operation Soroban transactions are also unaffected because the transaction-wide set and the per-operation set are identical.
