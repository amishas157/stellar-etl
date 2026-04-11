# H001: Pre-Network Time Ranges Skip Genesis Ledger

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`get_ledger_range_from_times` should return a range that includes ledger `1` when the requested start time predates network launch, matching the command text that says pre-network ranges "will start with the genesis ledger." Downstream exports that use this range should therefore include genesis-ledger rows instead of silently starting at ledger `2`.

## Mechanism

`GetLedgerRange()` hard-codes the searchable lower bound to ledger `2`: `createNewGraph()` sets `BeginPoint` from `getGraphPoint(2)`, `limitLedgerRange()` clamps out-of-range timestamps up to that point, and both `findLedgerForDate()` calls start from `2`. As a result, any request that should include the genesis ledger is normalized to ledger `2`, so the command emits a plausible-looking range that silently omits the first ledger from every downstream export seeded from it.

## Trigger

Run `stellar-etl get_ledger_range_from_times --start-time <time before network start> --end-time <time at or after network start>`. The correct JSON should begin with `"start":1`, but the current code path will return `"start":2`.

## Target Code

- `internal/input/ledger_range.go:GetLedgerRange:31-67` — clamps the requested timestamps and starts both searches from ledger `2`
- `internal/input/ledger_range.go:createNewGraph:70-98` — hard-codes `BeginPoint` to ledger `2`
- `internal/input/ledger_range.go:limitLedgerRange:196-210` — rewrites pre-network times to the close time of ledger `2`
- `cmd/get_ledger_range_from_times.go:18-82` — exposes the returned range directly to users as JSON

## Evidence

The only special-case handling for the genesis ledger is a comment saying ledger `1` has a Unix close time of `0`, but there is no compensating branch that maps pre-network requests back to `1`. The command help text and README both promise genesis-inclusive behavior, so returning `2` is not just an internal convention mismatch; it changes the exported range boundary.

## Anti-Evidence

The code deliberately avoids using ledger `1` in its time search because the genesis close time is not representative of network start time. If downstream users intentionally want "first ledger with a real close timestamp" instead of "full history," this behavior may have been convenient, but it still contradicts the command contract and drops ledger `1` from the returned range.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full execution path from `GetLedgerRange()` through `createNewGraph()`, `limitLedgerRange()`, `findLedgerForDate()`, and `findLedgerForTimeBinary()`. Confirmed that ledger 2 is used as the lower bound throughout the time-based search. Also examined the README, the `--start-ledger` flag definitions in `internal/utils/main.go`, and the command help text to determine whether this is a contract violation or deliberate design.

### Code Paths Examined

- `internal/input/ledger_range.go:GetLedgerRange:31-67` — starts both searches from ledger 2 with explicit comment explaining rationale (ledger 1 close time = Unix epoch 0)
- `internal/input/ledger_range.go:createNewGraph:70-98` — sets `BeginPoint` to ledger 2 with comment about genesis ledger's 1970s close time
- `internal/input/ledger_range.go:limitLedgerRange:196-210` — clamps pre-network start times to `BeginPoint.CloseTime` (ledger 2's close time)
- `internal/input/ledger_range.go:findLedgerForDate:143-193` — line 177 clamps to `BeginPoint.Seq` to avoid infinite cycles; binary search in `findLedgerForTimeBinary` also guards `end.Seq >= 2`
- `internal/utils/main.go:251,276` — `--start-ledger` flag defaults to `2`, described as "Defaults to genesis ledger"
- `internal/utils/main.go:738` — separate validation says "genesis ledger is ledger 1" but this is for the direct `--start-ledger` flag, not the time-based API
- `cmd/get_ledger_range_from_times.go:24-25` — help text says "the ledger range will start with the genesis ledger"

### Why It Failed

This is **working-as-designed behavior**, not a bug. The code deliberately excludes ledger 1 from time-based searches for a sound algorithmic reason: ledger 1's close time is Unix epoch 0 (January 1, 1970), which would break the binary search algorithm that assumes monotonically increasing close times from the begin point. The codebase consistently treats ledger 2 as the effective starting point for time-based operations:

1. The `--start-ledger` flag defaults to `2` across all export commands and calls this the "genesis" default (lines 251, 276 of `utils/main.go`).
2. The code contains explicit comments at lines 55-56 and 80 of `ledger_range.go` explaining the rationale.
3. Including ledger 1 would require special-case logic outside the binary search, and its data (initial state with no transactions) has minimal practical value for time-range exports.

The help text's use of "genesis ledger" is slightly ambiguous documentation, but the underlying behavior is intentional and correct. A documentation clarification (e.g., "will start with ledger 2, the first ledger with a valid close time") would resolve any confusion, but the code itself is not producing incorrect data.

### Lesson Learned

When the codebase has explicit comments explaining *why* a boundary value was chosen, treat the behavior as deliberate design unless there is concrete evidence of data corruption downstream. Ambiguous help text describing an intentional design choice is a documentation issue, not a data correctness bug.
