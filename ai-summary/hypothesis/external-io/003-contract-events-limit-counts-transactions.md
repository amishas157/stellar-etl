# H003: `export_contract_events --limit` counts transactions, not emitted event rows

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

The shared `--limit` flag is registered as "Maximum number of contract_events to export", so `export_contract_events --limit N` should serialize at most `N` contract-event rows.

## Mechanism

The command passes `cmdArgs.Limit` straight into `input.GetTransactions()`, which enforces the cap on transaction count. `TransformContractEvent()` then expands each selected transaction into the concatenation of `TransactionEvents`, `OperationEvents`, and `DiagnosticEvents`, and the command writes every returned row without any second event-level cap. A single Soroban transaction with multiple emitted events can therefore exceed the documented export limit all by itself.

## Trigger

Run `export_contract_events --limit 1` on a ledger containing one successful Soroban transaction that emits multiple transaction, operation, or diagnostic events. The CLI should stop after one contract-event row, but this path should export every event emitted by that one limited transaction.

## Target Code

- `internal/utils/main.go:248-255` - shared archive flag text promises a maximum number of `contract_events`
- `cmd/export_contract_events.go:25-65` - command applies `limit` to `GetTransactions()` and then writes all rows returned by `TransformContractEvent()`
- `internal/input/transactions.go:22-70` - transaction reader enforces the limit on `len(txSlice)`
- `internal/transform/contract_events.go:21-67` - one transaction expands into rows from `TransactionEvents`, `OperationEvents`, and `DiagnosticEvents`

## Evidence

`TransformContractEvent()` is explicitly one-to-many: it appends rows from three event collections per transaction. `export_contract_events.go` never tracks how many contract-event rows have already been emitted, so the only effective cap is the upstream transaction count.

## Anti-Evidence

Transactions that emit zero or one contract event will appear to honor the flag. The mismatch only surfaces when a limited transaction produces multiple exported rows.
