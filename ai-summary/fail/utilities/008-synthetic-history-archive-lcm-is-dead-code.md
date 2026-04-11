# H008: Synthetic history-archive `LedgerCloseMeta` builder is dead code on current exports

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the utilities layer synthesizes `LedgerCloseMeta` from history-archive ledger files, an active export path should consume that synthetic close meta correctly enough that missing fields could affect production output. A metadata-poor synthetic LCM would then be a concrete corruption risk.

## Mechanism

`historyArchiveBackend.GetLedger()` fabricates a V0 `LedgerCloseMeta` with only a header, transaction set, and result pairs. I suspected this stripped-down object might silently zero fee/meta-derived fields for any export path that passed it into `ingest.NewLedgerTransactionReaderFromLedgerCloseMeta()` or another LCM-based transform.

## Trigger

Exercise a production export path that uses `utils.CreateBackend(...).GetLedger(...)` rather than the original history-archive ledger object.

## Target Code

- `internal/utils/main.go:historyArchiveBackend.GetLedger:702-720` — synthesizes a minimal V0 `LedgerCloseMeta`
- `internal/input/ledgers_history_archive.go:GetLedgersHistoryArchive:10-25` — current history-archive ledger reader uses `GetLedgerArchive()`, not `GetLedger()`
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:13-39` — current history-archive asset reader also uses `GetLedgerArchive()` and never touches the synthetic LCM builder

## Evidence

The synthetic builder omits large parts of real close-meta structure, including transaction meta and fee-related change sets. If a live exporter consumed this helper, any downstream logic that expected full close meta would be at risk.

## Anti-Evidence

I did not find a current export path in this repository that actually calls `historyArchiveBackend.GetLedger()`. The existing `CreateBackend()` consumers use `GetLedgerArchive()` directly, and the already-known history-archive bugs in ledgers/assets do not depend on this synthetic LCM method.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The code looks dangerous in isolation, but it is not currently wired into a production export path. Without a live caller, the stripped-down synthetic `LedgerCloseMeta` does not presently qualify as a concrete ETL correctness bug.

### Lesson Learned

Synthetic helper outputs are only actionable findings when a real export path consumes them. For history-archive code in this repo, check whether callers use the fabricated close meta or the original archive ledger object before treating a lossy reconstruction as a live bug.
