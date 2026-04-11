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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced both `parseAssetBalanceChangesFromContractEvents` implementations (standalone at line 1942 and method at line 1907) through `GetDiagnosticEvents()` into the SDK's `TransactionMeta.GetDiagnosticEvents()`. In TxMeta V3, this reads `SorobanMeta.DiagnosticEvents` which is a SEPARATE XDR field from `SorobanMeta.Events` (the canonical contract events). In V4, it reads `txMeta.DiagnosticEvents` which is separate from `txMeta.Operations[i].Events`. The SDK comment at both the XDR and ingest layers explicitly warns that contract events appear in the diagnostic events list only conditionally depending on stellar-core's `ENABLE_SOROBAN_DIAGNOSTIC_EVENTS` config. The ETL's own `TransformContractEvent()` already uses the recommended `GetTransactionEvents()` API, confirming the codebase is aware of the correct approach.

### Code Paths Examined

- `internal/transform/operation.go:1063-1097` — `extractOperationDetails()` calls standalone `parseAssetBalanceChangesFromContractEvents(transaction, network)` for invoke_contract ops
- `internal/transform/operation.go:1715-1719` — `transactionOperationWrapper.Details()` calls method version `operation.parseAssetBalanceChangesFromContractEvents()`
- `internal/transform/operation.go:1907-1940` — method version: calls `operation.transaction.GetDiagnosticEvents()`, feeds through `filterEvents()`, extracts SAC balance changes
- `internal/transform/operation.go:1942-1975` — standalone version: identical logic, same `GetDiagnosticEvents()` dependency
- `internal/transform/operation.go:1886-1895` — `filterEvents()` selects `InSuccessfulContractCall=true && Type=ContractEventTypeContract`, confirming it expects contract events wrapped as diagnostic events
- `go-stellar-sdk/xdr/transaction_meta.go:35-49` — V3: returns `SorobanMeta.DiagnosticEvents`; V4: returns `txMeta.DiagnosticEvents`. Comment warns MAY include contract events depending on config.
- `go-stellar-sdk/ingest/ledger_transaction.go:260-266` — SDK-level wrapper with same warning recommending `GetTransactionEvents()` instead
- `internal/transform/contract_events.go:21-23` — `TransformContractEvent()` uses `GetTransactionEvents()` (the correct API), showing the codebase already has the right pattern
- `docker/stellar-core.cfg:5`, `docker/stellar-core_testnet.cfg:8`, `docker/stellar-core_futurenet.cfg:9` — all bundled configs have `ENABLE_SOROBAN_DIAGNOSTIC_EVENTS=true`, masking the bug in standard deployment

### Findings

The finding is real: `parseAssetBalanceChangesFromContractEvents()` uses `GetDiagnosticEvents()` which only contains contract events when `ENABLE_SOROBAN_DIAGNOSTIC_EVENTS=true` in the stellar-core config. The canonical contract events live in `SorobanMeta.Events` (V3) and `Operations[i].Events` (V4), accessible via `GetContractEvents()` or `GetTransactionEvents()`.

**Why Medium, not High:** The ETL's bundled configs all enable diagnostic events, so standard deployments produce correct output. The bug manifests only when:
1. A user provides a custom `--core-config` without `ENABLE_SOROBAN_DIAGNOSTIC_EVENTS=true`
2. The ETL processes data from history archives produced by nodes with different config
3. Future protocol changes alter V4 diagnostic event inclusion behavior

The failure mode is silent: `asset_balance_changes` would be an empty array (no error), making it a data completeness issue that's difficult to detect downstream.

**Inconsistency with sibling code:** `TransformContractEvent()` at `contract_events.go:22-23` uses `transaction.GetTransactionEvents()`, demonstrating the codebase already uses the recommended API elsewhere. The `parseAssetBalanceChangesFromContractEvents()` functions are outliers using the deprecated pattern.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Construct an `ingest.LedgerTransaction` with V3 TxMeta where `SorobanMeta.Events` contains a SAC transfer `ContractEvent` but `SorobanMeta.DiagnosticEvents` is empty (simulating `ENABLE_SOROBAN_DIAGNOSTIC_EVENTS=false`). Wire it into a `transactionOperationWrapper` for an `InvokeHostFunction` op of type `invoke_contract`.
- **Steps**: Call `parseAssetBalanceChangesFromContractEvents()` (both the standalone and method versions)
- **Assertion**: Assert that `asset_balance_changes` is empty (`len(balanceChanges) == 0`) when contract events are NOT duplicated into diagnostic events. Then demonstrate the fix: using `GetContractEvents()` or `GetTransactionEvents().OperationEvents` should return the SAC events regardless of diagnostic event config.
