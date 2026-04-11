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
