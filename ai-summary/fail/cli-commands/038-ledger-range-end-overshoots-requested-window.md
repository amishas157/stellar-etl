# H002: `get_ledger_range_from_times` returns an end ledger after the requested `--end-time`

**Date**: 2026-04-15
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Because the export commands consume `end-ledger` inclusively, the utility that prepares those ranges should return the last ledger whose `closed_at` is less than or equal to the requested `--end-time`. Otherwise, downstream exports include rows that fall outside the requested time window.

## Mechanism

`GetLedgerRange` uses the same `findLedgerForDate` helper for both the start and end boundaries. That helper returns the first ledger whose close time is greater than or equal to the target timestamp, which is correct for `start-time` but one ledger too far for an inclusive `end-ledger`; the resulting JSON range therefore causes `export_ledgers`, `export_transactions`, and sibling commands to over-export the first ledger closed after the requested end time.

## Trigger

Choose any `--end-time` that falls strictly between two ledger close times, run `get_ledger_range_from_times`, and feed the returned JSON into an inclusive export command. Example probe: `2019-09-14T13:35:10Z` returns `end=25821065`, but ledger `25821065` closes at `13:35:11Z` while ledger `25821064` closes at `13:35:06Z`.

## Target Code

- `cmd/get_ledger_range_from_times.go:Run:63-80` — emits the `start`/`end` pair returned by `input.GetLedgerRange(...)`
- `internal/input/ledger_range.go:GetLedgerRange:31-67` — computes both boundaries with the same `findLedgerForDate(...)` helper
- `internal/input/ledger_range.go:findLedgerForDate:142-193` — returns the first ledger on or after the target timestamp

## Evidence

The README says this utility "aids in the usage of Export Commands" and separately documents that export-command ledger ranges are inclusive. A live probe against the current code returned `start=25811356 end=25821065` for `2019-09-13T23:00:00Z` to `2019-09-14T13:35:10Z`; the previous ledger `25821064` closes at `13:35:06Z`, while returned ledger `25821065` closes at `13:35:11Z`, outside the requested window.

## Anti-Evidence

The README also says the helper returns the smallest ledger range that "completely covers" the provided period, which could be read as intentionally including the first post-window ledger. This hypothesis depends on the helper's practical contract being "prepare inclusive export-command bounds," not "cover wall-clock time by closure intervals."

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `GetLedgerRange` through both `findLedgerForDate` (recursive, lines 142-194) and `findLedgerForTimeBinary` (binary search, lines 101-140). Both use the identical termination condition: `prevTime.CloseTime.Unix() < targetTime.Unix() && currentPoint.CloseTime.Unix() >= targetTime.Unix()`, returning the first ledger whose close time is ≥ the target. This is applied identically to both start and end boundaries. The README (line 333) explicitly documents this as returning "the smallest possible ledger range that completely covers the provided time period."

### Code Paths Examined

- `internal/input/ledger_range.go:GetLedgerRange:32-67` — calls `findLedgerForDate` identically for both start and end, no post-adjustment
- `internal/input/ledger_range.go:findLedgerForDate:142-194` — termination at line 158: returns first ledger with closeTime ≥ target
- `internal/input/ledger_range.go:findLedgerForTimeBinary:101-140` — identical termination at line 117
- `README.md:333` — "The ledger range that is returned will be the smallest possible ledger range that completely covers the provided time period"

### Why It Failed

The behavior is **working as designed** per the explicitly documented contract. The README states the command returns the "smallest possible ledger range that completely covers the provided time period." In Stellar, a ledger captures all transactions between the previous ledger's close and its own close. If a user's end-time falls between ledger N's close (T1) and ledger N+1's close (T2), then transactions occurring between T1 and the user's end-time are recorded in ledger N+1. Excluding N+1 would **lose data** that occurred within the requested time window. The hypothesis's expected behavior ("return the last ledger whose closed_at ≤ end-time") would actually introduce data loss, not fix a bug.

### Lesson Learned

When a utility documents a "complete coverage" contract, the end-boundary overshoot is intentional — it ensures no transactions within the requested window are missed due to ledger-boundary granularity. The anti-evidence section of the hypothesis correctly identified this possibility but the hypothesis was filed anyway.
