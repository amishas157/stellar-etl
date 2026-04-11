# H001: CAP-67 invoke-host-function balance changes read the wrong event stream

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an `InvokeHostFunction` operation emits Stellar Asset Contract transfer, mint, burn, or clawback events, `history_operations.details.asset_balance_changes` should contain one entry per emitted balance delta with the correct `from`, `to`, `amount`, and asset fields for that operation. This should hold for CAP-67 / `TransactionMetaV4` exports where contract events are stored in per-operation metadata.

## Mechanism

`extractOperationDetails()` fills `asset_balance_changes` by calling `parseAssetBalanceChangesFromContractEvents()`, but that helper reads `transaction.GetDiagnosticEvents()` and then filters those diagnostics back into contract events. In `TransactionMetaV4`, the SDK documents that real contract events live in `OperationEvents`, while diagnostics are a separate stream that may not contain those contract events at all. On ledgers where diagnostics are present without mirrored contract events, the ETL silently exports `asset_balance_changes: []` even though the operation emitted real SAC balance-change events.

## Trigger

Export operations for a ledger containing a CAP-67 / `TransactionMetaV4` `InvokeHostFunction` call into the Stellar Asset Contract (`transfer`, `mint`, `burn`, or `clawback`) where txmeta diagnostics do not duplicate contract events. Compare the exported operation row against the operation-scoped contract events for the same transaction: the contract events show the asset movement, but `details.asset_balance_changes` is empty.

## Target Code

- `internal/transform/operation.go:1093-1097` — `extractOperationDetails()` populates `details["asset_balance_changes"]` from the helper
- `internal/transform/operation.go:1942-1975` — `parseAssetBalanceChangesFromContractEvents()` reads only `GetDiagnosticEvents()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/ingest/ledger_transaction.go:260-275` — SDK documents diagnostics as a separate stream and says V4 contract events live in `OperationEvents`

## Evidence

The live helper never asks for per-operation contract events; it only walks filtered diagnostics. The SDK comment for `GetTransactionEvents()` explicitly says CAP-67 moved contract events into `OperationMetaV2.Events`, and `GetDiagnosticEvents()` warns that diagnostic streams may or may not include contract events depending on txmeta generation. Current tests only pin a V3 single-operation path and already expect `asset_balance_changes` to be empty, so this V4 contract-event path is unexercised.

## Anti-Evidence

`GetDiagnosticEvents()` warns that some txmeta generation modes may mirror contract events into diagnostics. In those environments the bug can be masked because the helper accidentally rediscovers the needed SAC events from the duplicated diagnostic stream.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced the full execution path from `extractOperationDetails()` through `parseAssetBalanceChangesFromContractEvents()` into the SDK's `GetDiagnosticEvents()` method, and confirmed that for `TransactionMetaV4` (Protocol 23+), `GetDiagnosticEvents()` returns `txMeta.DiagnosticEvents` — a separate transaction-level stream that the SDK documents as conditionally containing contract events ("MAY include"). The correct per-operation contract events live in `txMeta.Operations[i].Events`, accessible via `GetContractEventsForOperation(opIndex)`. Notably, the same codebase's `effects.go:122` correctly uses `GetContractEvents()` (which delegates to `GetContractEventsForOperation(0)`) for the equivalent purpose, confirming the intended API.

### Code Paths Examined

- `internal/transform/operation.go:1063-1097` — `extractOperationDetails()` for `InvokeHostFunction` calls standalone `parseAssetBalanceChangesFromContractEvents(transaction, network)` and assigns the result to `details["asset_balance_changes"]`
- `internal/transform/operation.go:1942-1975` — standalone `parseAssetBalanceChangesFromContractEvents()` calls `transaction.GetDiagnosticEvents()`, filters via `filterEvents()`, and parses SAC events from the filtered diagnostic stream
- `internal/transform/operation.go:1886-1895` — `filterEvents()` keeps only `InSuccessfulContractCall && ContractEventTypeContract` events from the diagnostic stream
- `internal/transform/operation.go:1907-1940` — method version `(operation *transactionOperationWrapper) parseAssetBalanceChangesFromContractEvents()` has the same issue but appears unused from `extractOperationDetails`
- `stellar/go ingest/ledger_transaction.go:260-266` — `GetDiagnosticEvents()` delegates to `UnsafeMeta.GetDiagnosticEvents()`
- `stellar/go xdr/transaction_meta.go:35-50` — `TransactionMeta.GetDiagnosticEvents()` for V4 returns `t.MustV4().DiagnosticEvents` — a separate field from `Operations[i].Events`
- `stellar/go xdr/transaction_meta.go:7-29` — `GetContractEventsForOperation()` for V4 returns `txMeta.Operations[opIndex].Events` — the correct per-operation contract events
- `stellar/go ingest/ledger_transaction.go:245-258` — `GetContractEvents()` calls `GetContractEventsForOperation(0)` for soroban transactions
- `stellar/go ingest/ledger_transaction.go:268-313` — `GetTransactionEvents()` V4 case explicitly separates `OperationEvents` (from `Operations[i].Events`) from `DiagnosticEvents` (from `txMeta.DiagnosticEvents`)
- `internal/transform/effects.go:118-130` — `effects.go` correctly uses `GetContractEvents()` for `InvokeHostFunction` effects, confirming the intended API usage
- `internal/transform/transaction.go:195-211` — confirms V4 meta is actively handled in the codebase for P23+ ledgers

### Findings

**The bug is confirmed.** `parseAssetBalanceChangesFromContractEvents()` reads SAC balance-change events from `GetDiagnosticEvents()`, which in `TransactionMetaV4` returns the `DiagnosticEvents` field — a separate stream from per-operation contract events. The SDK explicitly documents:

1. `GetDiagnosticEvents()` comment (line 260-263): "it is possible that, for smart contract transactions, the list of generated diagnostic events **MAY** include contract events as well" — meaning inclusion is not guaranteed.

2. `GetTransactionEvents()` comment (line 268-276): "Contract events will now be present in the 'operation []OperationMetaV2' in the TransactionMetaV4 structure, instead of at the transaction level as in TxMetaV3."

3. The V4 `GetTransactionEvents()` implementation (lines 300-307) stores per-operation contract events in `OperationEvents[i]` and diagnostic events separately in `DiagnosticEvents`.

The correct API is `GetContractEventsForOperation(opIndex)` or `GetContractEvents()` (for soroban transactions with a single operation). The `effects.go` code at line 122 already uses this correct API.

**Severity rationale: High** (downgraded from Critical). The impact is empty `asset_balance_changes` when SAC events occurred — "empty results from broken control flow" (High) rather than "wrong numeric value" (Critical). The data is silently absent rather than numerically incorrect. The manifestation depends on whether the stellar-core txmeta generation configuration mirrors contract events into diagnostics: if mirrored, the bug is masked; if not mirrored, the field is silently empty.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Construct an `ingest.LedgerTransaction` with `UnsafeMeta.V = 4` and a `TransactionMetaV4` where:
  - `Operations[0].Events` contains a SAC transfer `ContractEvent` (type `Contract`, with `InSuccessfulContractCall = true` inside a wrapping `DiagnosticEvent` if needed — but critically, place the contract event in the per-operation events list)
  - `DiagnosticEvents` is empty (or contains only non-contract diagnostic events)
  - The transaction envelope wraps an `InvokeHostFunction` operation invoking the SAC
- **Steps**: Call `parseAssetBalanceChangesFromContractEvents(transaction, network)` with this V4 transaction
- **Assertion**: Assert that `balanceChanges` is empty (demonstrating the bug). Then, as a positive control, call `transaction.GetContractEventsForOperation(0)` and assert it returns the SAC event — proving the event is present but the function reads from the wrong stream
