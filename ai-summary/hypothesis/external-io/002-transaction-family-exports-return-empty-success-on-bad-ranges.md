# H002: Transaction-family exports can return empty success on bad ledger ranges

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_transactions`, `export_operations`, `export_effects`, `export_contract_events`, `export_trades`, and `export_ledger_transaction` should fail fast when the requested bounded range is impossible, instead of emitting a plausible empty export that downstream jobs can mistake for “no rows in this range”.

## Mechanism

`GetTransactions()`, `GetOperations()`, and `GetTrades()` each call `CreateLedgerBackend()` and `PrepareRange(BoundedRange(start, end))` without any local `ValidateLedgerRange()` check. If the backend accepts the bound object, the outer `for seq := start; seq <= end; seq++` loop is never entered for `end == 0` or `end < start`, so the commands receive empty slices and complete normally with zero exported rows.

## Trigger

1. Run any transaction-derived export with an impossible range, for example `stellar-etl export_transactions --start-ledger 100 --end-ledger 50 -o tx.json` or `stellar-etl export_trades --start-ledger 2 --end-ledger 0 -o trades.json`.
2. The command is expected to reject the invalid range, but may instead produce an empty JSON file and optional empty Parquet artifact.

## Target Code

- `internal/input/transactions.go:GetTransactions:23-70` — no range validation before the outer ledger loop
- `internal/input/operations.go:GetOperations:30-80` — same skipped-loop structure for operation exports
- `internal/input/trades.go:GetTrades:26-85` — same skipped-loop structure for trade exports
- `cmd/export_transactions.go:Run:17-66` — treats empty transaction input as success
- `cmd/export_effects.go:Run:17-69` — derives effects from `GetTransactions()` and will happily emit zero rows
- `cmd/export_contract_events.go:Run:17-66` — same `GetTransactions()` empty-success path
- `cmd/export_trades.go:Run:20-71` — same empty-success path for `GetTrades()`
- `cmd/export_ledger_transaction.go:Run:17-57` — same `GetTransactions()` path for ledger-transaction exports

## Evidence

Unlike `utils.CreateBackend()` and `input.PrepareCaptiveCore()`, these helpers never call `ValidateLedgerRange()`. All affected commands treat a zero-length input slice as ordinary success, so there is no later guard that turns an impossible request into an error once the read helper returns empty.

## Anti-Evidence

This depends on the active `ledgerbackend` implementation continuing to accept `BoundedRange(start, end)` with bad bounds during `PrepareRange()`. If a future backend starts rejecting these ranges eagerly, the commands would fail before the silent-empty path is reached.
