# H033: Ignored non-EOF transaction-reader errors can corrupt input slices

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

The transaction-based input readers should either return a hard error or stop cleanly when `txReader.Read()` fails. They should not continue building export inputs from a failed read when processing legitimate Stellar ledger data.

## Mechanism

`GetTransactions()`, `GetOperations()`, `GetTrades()`, and `GetAllHistory()` only special-case `io.EOF` after `txReader.Read()`. At first glance, that looks like a silent-corruption path: a non-EOF error could leave `tx` at its zero value and still let the readers append bogus transactions, operations, or trades into the export slices.

## Trigger

Feed a ledger close meta into the readers such that `ingest.LedgerTransactionReader.Read()` returns a non-EOF error.

## Target Code

- `internal/input/transactions.go:GetTransactions:51-62` — ignores non-EOF `Read()` errors and appends `tx`
- `internal/input/operations.go:GetOperations:52-71` — same pattern before expanding operations
- `internal/input/trades.go:GetTrades:50-77` — same pattern before expanding trade-capable ops
- `internal/input/all_history.go:GetAllHistory:55-80` — same pattern in the combined reader

## Evidence

Each reader checks only `if err == io.EOF { break }` and otherwise continues to use the returned `tx` value. There is no `if err != nil { return ... }` guard on these loops.

## Anti-Evidence

In the current SDK version, the concrete non-EOF `LedgerTransactionReader.Read()` failures I traced are driven by malformed or internally inconsistent `LedgerCloseMeta` (for example, transaction-result hashes that do not match the envelope list), not by an otherwise legitimate Stellar transaction flowing through a healthy backend. That makes the suspected corruption path depend on bad upstream metadata rather than the “legitimate transaction” trigger required for this objective.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The live trigger is malformed backend ledger metadata, not a normal on-chain transaction shape. For this objective, that makes the issue an upstream-data-integrity or backend-corruption problem rather than a legitimate-transaction export bug.

### Lesson Learned

Suspicious ignored-error loops are only in-scope here when the underlying reader can fail on valid ledger contents. If the concrete non-EOF errors require broken `LedgerCloseMeta`, record the path as a dead end instead of forcing a data-correctness finding.
