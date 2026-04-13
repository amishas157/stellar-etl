# H003: `export_contract_events --limit` counts transactions, not emitted event rows

**Date**: 2026-04-13
**Subsystem**: external-io
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Because the shared flag text advertises `--limit` as the maximum number of exported `contract_events`, `export_contract_events --limit N` should emit at most `N` `ContractEventOutput` rows.

## Mechanism

The command passes `cmdArgs.Limit` into `input.GetTransactions()`, so the cap is applied to input transactions before any event expansion happens. `TransformContractEvent()` can emit rows from `TransactionEvents`, nested `OperationEvents`, and `DiagnosticEvents` for a single selected transaction, and `cmd/export_contract_events.go` writes the full returned slice with no secondary row cap.

## Trigger

Run `stellar-etl export_contract_events --limit 1` over any range whose first returned Soroban transaction emits more than one contract-event/diagnostic row.

## Target Code

- `internal/utils/main.go:250-254` — advertises `--limit` as the maximum number of `contract_events` to export
- `cmd/export_contract_events.go:25-43` — applies `cmdArgs.Limit` at `GetTransactions()` and then writes every row returned by `TransformContractEvent()`
- `internal/input/transactions.go:23-70` — enforces the limit on transaction count
- `internal/transform/contract_events.go:21-67` — expands one transaction into many `ContractEventOutput` rows across three event collections

## Evidence

`TransformContractEvent()` has three independent append loops and can therefore emit multiple rows for a single transaction even before considering multiple operation-scoped events. The command never tracks an emitted-row budget; it only iterates over the already-limited transaction slice and writes every transformed row.

## Anti-Evidence

Transactions with exactly one exported event will not expose the bug. This is a row-count contract violation rather than field-level corruption inside individual event rows.
