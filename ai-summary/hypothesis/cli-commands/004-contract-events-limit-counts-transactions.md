# H004: `export_contract_events --limit` caps transactions instead of contract-event rows

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural row-count corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an operator runs `export_contract_events --limit 1`, the command should emit at most one contract-event row, because the shared archive flag is documented as the maximum number of exported `contract_events`. A range-sized batch should not silently expand into hundreds of rows from the first transaction alone.

## Mechanism

`export_contract_events` passes `cmdArgs.Limit` into `input.GetTransactions()`, so the limit is enforced on transactions. `transform.TransformContractEvent()` then flattens a single transaction's `TransactionEvents`, `OperationEvents`, and `DiagnosticEvents` into one combined `[]ContractEventOutput`, and the command exports that whole slice with no row-level cap.

This makes `--limit` a hidden transaction limit rather than a contract-event limit. Smart-contract transactions with large diagnostic/event fan-out therefore produce success-shaped files containing vastly more rows than the CLI flag promised.

## Trigger

Run `export_contract_events --limit 1` over a range whose first returned transaction emits multiple contract or diagnostic events. The checked-in fixture for `cmd/export_contract_events_test.go` already demonstrates that such fan-out is common: grouping `testdata/contract_events/large_range_ledger_txs.golden` by `transaction_id` shows transaction `224503691523153920` alone contributes 137 exported rows.

## Target Code

- `internal/utils/main.go:AddArchiveFlags:250-254` — public flag contract says `limit` is the maximum number of exported `contract_events`
- `cmd/export_contract_events.go:19-25` — parses `cmdArgs.Limit` and forwards it into `GetTransactions()`
- `cmd/export_contract_events.go:33-53` — writes every event row returned from `TransformContractEvent()`
- `internal/input/transactions.go:GetTransactions:22-70` — enforces `limit` on transactions, not contract-event rows
- `internal/transform/contract_events.go:21-68` — expands one transaction into a combined slice of transaction, operation, and diagnostic event rows
- `cmd/export_contract_events_test.go:7-19` — existing end-to-end CLI coverage for this command path

## Evidence

`TransformContractEvent()` appends rows from three separate event collections for every transaction, so multi-row fan-out is inherent to the design. The checked-in golden fixture contains 13,035 rows across 3,915 transaction IDs, and some single transactions explode into triple-digit row counts; that proves a transaction-capped limit can materially overrun the exported row count without any error path firing.

## Anti-Evidence

Because the command is transaction-driven internally, maintainers may have intended `limit` to mean "transactions scanned." But the flag text comes from `AddArchiveFlags("contract_events", ...)`, and nothing in the command help warns users that the limit is applied before event expansion, so the current behavior still violates the exported-object contract.
