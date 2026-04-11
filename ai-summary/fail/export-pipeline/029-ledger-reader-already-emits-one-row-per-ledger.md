# H029: `GetLedgers()` duplicates `history_ledgers` rows once per transaction

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledgers` should emit one row per ledger sequence, even when a ledger
contains many transactions. A ledger with 50 transactions should still yield a
single `history_ledgers` row describing that ledger's aggregates.

## Mechanism

I initially suspected that `GetLedgers()` rebuilt a synthetic
`historyarchive.Ledger` from per-transaction structures and might append one
result per transaction instead of one result per sequence. That would have made
multi-transaction ledgers fan out into duplicate ledger rows with identical
header data.

## Trigger

Run `stellar-etl export_ledgers` over any ledger containing multiple
transactions and compare the number of exported rows to the number of ledger
sequences in the requested range.

## Target Code

- `internal/input/ledgers.go:21-90` — reconstructs a `historyarchive.Ledger` from each fetched `LedgerCloseMeta`
- `cmd/export_ledgers.go:25-44` — exports exactly one transformed row for each element returned by `GetLedgers()`
- `internal/transform/ledger.go:16-130` — consumes a whole-ledger structure and computes aggregate counts from its transaction set

## Evidence

`GetLedgers()` does substantial work to rebuild a transaction set and result set
for each fetched ledger, which made it plausible that the returned slice was
transaction-shaped rather than ledger-shaped.

## Anti-Evidence

The append site is outside every transaction loop: `GetLedgers()` constructs one
`ledgerLCM` per `seq` and appends it exactly once before advancing to the next
ledger. `cmd/export_ledgers.go` then iterates that slice 1:1, so there is no
fan-out path that could duplicate a ledger row per transaction.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The returned slice is already ledger-granular. `GetLedgers()` reconstructs the
full transaction/result payload for counting purposes, but it still appends only
one `HistoryArchiveLedgerAndLCM` per ledger sequence.

### Lesson Learned

Rebuilt ledger-wide aggregates can look transaction-shaped without actually
changing row cardinality. Before filing a duplication hypothesis, trace the
exact append site and the caller's iteration unit.
