# H001: `invoke_contract` asset balance changes depend on diagnostic-event duplication

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: operation detail corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a successful `invoke_contract` operation that emits Stellar Asset Contract transfer/mint/burn/clawback events, `history_operations.details.asset_balance_changes` should be derived from the operation's authoritative contract-event stream. The exported balance-change rows should be present regardless of whether the transaction metadata generator duplicates those contract events into the diagnostic-event list.

## Mechanism

`extractOperationDetails()` populates `asset_balance_changes` through `parseAssetBalanceChangesFromContractEvents()`, but that helper reads only `transaction.GetDiagnosticEvents()`. The upstream ingest API explicitly warns that diagnostic events may or may not include contract events depending on txmeta generation configuration, while `GetTransactionEvents()` exposes the canonical operation-scoped contract events. The same valid SAC transaction can therefore export populated balance changes under one metadata mode and an empty list under another.

## Trigger

Export a successful Soroban `invoke_contract` transaction that emits SAC transfer-like contract events, using TxMeta where the operation events are present but the diagnostic-event list does not duplicate those contract events. The operation row will contain the correct `contract_id` and parameters, but `details.asset_balance_changes` will be empty or incomplete.

## Target Code

- `internal/transform/operation.go:1063-1097` — `extractOperationDetails()` wires `invoke_contract` details to `parseAssetBalanceChangesFromContractEvents()`
- `internal/transform/operation.go:1942-1975` — helper reads only `transaction.GetDiagnosticEvents()` and filters that stream
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:260-311` — SDK comments document that diagnostic events may include contract events only conditionally and recommend `GetTransactionEvents()`

## Evidence

The helper name says it parses balance changes "from contract events", but the implementation never reads operation events at all. The ingest SDK comment at `GetDiagnosticEvents()` says contract events MAY be present there depending on configuration, and `GetTransactionEvents()` separately materializes per-operation contract events in both V3 and V4 metadata layouts.

## Anti-Evidence

This bug is masked when the txmeta producer happens to include contract events in the diagnostic-event stream. Current Soroban transactions also have only one smart-contract operation, so the likely failure mode is omission/config-dependence rather than cross-operation mixing.
