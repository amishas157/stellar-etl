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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (same bug class as success #006 for effects, but distinct command)

### Trace Summary

The `export_contract_events` command registers `--limit` via `AddArchiveFlags("contract_events", ...)` which describes it as "Maximum number of contract_events to export." The limit value is passed directly to `input.GetTransactions()` at line 25, where `GetTransactions` enforces it against `len(txSlice)` (transaction count) at line 51 of `transactions.go`. The command then iterates all transactions and calls `TransformContractEvent()` which concatenates events from three separate sources (TransactionEvents, OperationEvents, DiagnosticEvents) into a single slice. Every returned event is written to output at lines 42–53 with no second row-level cap.

### Code Paths Examined

- `internal/utils/main.go:250-254` — `AddArchiveFlags` registers `--limit` with text "Maximum number of contract_events to export"
- `cmd/export_contract_events.go:25` — passes `cmdArgs.Limit` to `input.GetTransactions()`, limiting transaction count not event count
- `cmd/export_contract_events.go:33-54` — iterates ALL transactions, calls `TransformContractEvent()`, writes every event row — no event-level cap
- `internal/input/transactions.go:51` — `for int64(len(txSlice)) < limit || limit < 0` — limit controls transaction count
- `internal/input/transactions.go:65` — `if int64(len(txSlice)) >= limit && limit >= 0 { break }` — outer loop also exits on transaction count
- `internal/transform/contract_events.go:21-67` — `TransformContractEvent` is one-to-many: appends from TransactionEvents (line 31), OperationEvents (line 42), and DiagnosticEvents (line 58)

### Findings

The bug is structurally identical to confirmed success finding #006 (effects-limit-counts-transactions-not-effects) but in the `export_contract_events` command. The `--limit` flag's documented semantics promise a cap on contract event rows, but the implementation caps transaction count instead. A single Soroban transaction can emit events from three distinct sources (TransactionEvents, OperationEvents, DiagnosticEvents), so `--limit 1` could easily produce dozens of exported event rows.

The deviation from documented behavior is clear: the flag text says "contract_events" but the code limits transactions. This is an operational correctness issue — output row counts violate the documented export bound. Row content is correct; only the total count is wrong.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go` (or a new integration-level test)
- **Setup**: Construct a single `ingest.LedgerTransaction` with multiple events across TransactionEvents, OperationEvents, and/or DiagnosticEvents
- **Steps**: Call `TransformContractEvent()` on that single transaction and count the returned rows
- **Assertion**: Assert `len(result) > 1` — proving that one "limited" transaction expands into multiple event rows, exceeding the documented `--limit 1` contract
