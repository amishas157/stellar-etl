# H002: Transaction-family exports can return empty success on bad ledger ranges

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_transactions`, `export_operations`, `export_effects`, `export_contract_events`, `export_trades`, and `export_ledger_transaction` should fail fast when the requested bounded range is impossible, instead of emitting a plausible empty export that downstream jobs can mistake for "no rows in this range".

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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (H001 covers `GetLedgers()` in `export_ledgers`/`export_token_transfers`; this covers the distinct `GetTransactions()`/`GetOperations()`/`GetTrades()` family used by 6 other commands)

### Trace Summary

`GetTransactions()` (transactions.go:23), `GetOperations()` (operations.go:30), and `GetTrades()` (trades.go:26) all call `utils.CreateLedgerBackend()` which builds either a `CaptiveStellarCore` backend (via `CreateCaptiveCoreBackend()`, line 921) or a `BufferedStorageBackend` (line 1036) — neither path calls `ValidateLedgerRange()`. The subsequent `for seq := start; seq <= end; seq++` loop silently produces zero iterations when `end < start` or when `end == 0` (with default `start = 2`). The empty slice propagates to each export command, which writes an empty output file, closes it, and calls `MaybeUpload()` — potentially pushing an empty artifact to GCS. This contrasts with `StreamChanges()` (changes.go:68) and `CreateBackend()` (main.go:770), which both call `ValidateLedgerRange()`.

### Code Paths Examined

- `internal/input/transactions.go:GetTransactions:23-71` — calls `CreateLedgerBackend()` then iterates `for seq := start; seq <= end; seq++`; no `ValidateLedgerRange()` call; returns empty slice with nil error when loop is skipped
- `internal/input/operations.go:GetOperations:30-81` — identical pattern to `GetTransactions()`; no range validation
- `internal/input/trades.go:GetTrades:26-86` — identical pattern; no range validation
- `internal/utils/main.go:CreateLedgerBackend:1011-1041` — captive-core path calls `CreateCaptiveCoreBackend()` (line 921-943) which does NOT validate ranges; datastore path builds `BufferedStorageBackend` which also does NOT validate ranges
- `internal/utils/main.go:ValidateLedgerRange:736-758` — validates start≠0, end≠0, end≥start, range within latest ledger — exists but is NOT called by any of the three reader functions
- `internal/input/changes.go:68` — `StreamChanges` DOES call `ValidateLedgerRange()`, confirming the validation is expected for bounded range reads
- `internal/utils/main.go:CreateBackend:760-770` — the history-archive path also calls `ValidateLedgerRange()`, further confirming the pattern
- `cmd/export_transactions.go:25-28` — calls `GetTransactions(startNum, commonArgs.EndNum, ...)`, iterates result, closes file, calls `MaybeUpload()` — no guard for empty result
- `cmd/export_operations.go:25-28` — same pattern with `GetOperations()`
- `cmd/export_effects.go:25-28` — same pattern, deriving effects from `GetTransactions()` output
- `cmd/export_trades.go:28-31` — same pattern with `GetTrades()`
- `cmd/export_contract_events.go:25-28` — same pattern with `GetTransactions()`
- `cmd/export_ledger_transaction.go:25-28` — same pattern with `GetTransactions()`
- `internal/utils/main.go:MustArchiveFlags:541-562` — parses `start-ledger` (default 2) with no validation
- `internal/utils/main.go:MustCommonFlags:460-538` — parses `end-ledger` (default 0) with no validation

### Findings

The bug is confirmed across all 6 affected commands. Key observations:

1. **No validation in either backend path**: Unlike H001 where the captive-core path through `CreateBackend()` validates ranges, the `CreateLedgerBackend()` function used by `GetTransactions()`/`GetOperations()`/`GetTrades()` performs NO validation in either the captive-core or datastore paths. This means there is no configuration that enables validation for these commands.

2. **Wider blast radius than H001**: This affects 6 commands (`export_transactions`, `export_operations`, `export_effects`, `export_contract_events`, `export_trades`, `export_ledger_transaction`) vs H001's 2 commands (`export_ledgers`, `export_token_transfers`).

3. **Empty files can upload to GCS**: All 6 commands call `MaybeUpload()` after the export loop, so an empty file can be pushed to cloud storage, creating silent data gaps in downstream pipelines.

4. **Inconsistency is clear**: `StreamChanges()` and `CreateBackend()` both call `ValidateLedgerRange()`, but `GetTransactions()`, `GetOperations()`, and `GetTrades()` do not. The codebase has the validation function; these paths just don't use it.

### PoC Guidance

- **Test file**: `cmd/export_transactions_test.go` (if it exists; otherwise create alongside existing test files)
- **Setup**: The existing test infrastructure builds the binary and runs CLI commands. Mirror the pattern in `cmd/export_ledgers_test.go` if available.
- **Steps**:
  1. Run `export_transactions --start-ledger 100 --end-ledger 50 -o tx.json` and verify it produces an empty output rather than an error
  2. Run `export_operations --start-ledger 100 --end-ledger 50 -o ops.json` and verify the same silent empty behavior
  3. Run `export_trades --start-ledger 100 --end-ledger 50 -o trades.json` and verify the same
- **Assertion**: Current behavior produces "Number of bytes written: 0" with a success exit code. The fix would add `ValidateLedgerRange()` calls in `GetTransactions()`, `GetOperations()`, and `GetTrades()` (or at the command level before calling them), causing these cases to return an error. A simpler fix: add a local `if end < start || end == 0 { return error }` check at the top of each function.
