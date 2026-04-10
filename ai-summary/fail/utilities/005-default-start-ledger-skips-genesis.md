# H003: Default start-ledger of 2 silently skips genesis-ledger exports

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When export commands say `start-ledger` defaults to the genesis ledger, omitting that flag should make the ETL start at ledger `1`. The exported ledger, transaction, operation, effect, trade, and ledger-entry-change datasets should therefore include ledger `1` whenever the user does not override the range start.

## Mechanism

Both `AddArchiveFlags` and `AddCoreFlags` register `start-ledger` with default `2`, but their descriptions say the default is the genesis ledger. `MustArchiveFlags` and `MustCoreFlags` return that default unchanged, and the input readers iterate from `start` inclusively. As a result, a default export silently drops ledger `1` and any genesis-era records while still presenting the range as a normal successful run.

## Trigger

Invoke an archive-based export with no explicit start, for example `stellar-etl export_ledgers --end-ledger 10` or `stellar-etl export_transactions --end-ledger 10`. The command starts reading at ledger `2`, so ledger `1` never appears in the output even though the flag description says the default is genesis.

## Target Code

- `internal/utils/main.go:AddArchiveFlags:250-255` — sets `start-ledger` default to `2`
- `internal/utils/main.go:AddCoreFlags:267-277` — repeats the same default for core-based exports
- `internal/utils/main.go:MustArchiveFlags:541-562` — passes the default through unchanged
- `internal/input/ledgers.go:GetLedgers:14-25` — iterates from `start` inclusively
- `internal/input/transactions.go:GetTransactions:22-38` — iterates from `start` inclusively

## Evidence

The flag builders use `flags.Uint32P("start-ledger", "s", 2, ...)` while the help text says "Defaults to genesis ledger," and multiple command paths call `MustArchiveFlags`/`MustCoreFlags` directly before handing the result to inclusive `for seq := start; seq <= end; seq++` loops. The README repeats the same contradiction by documenting "Defaults to genesis ledger" alongside a default value of `2`.

## Anti-Evidence

`internal/input/ledger_range.go` intentionally starts time-to-ledger searches at ledger `2` because ledger `1` has Unix close time `0`. That rationale is specific to date-range conversion, though, and it does not justify silently excluding ledger `1` from normal export commands that work on explicit ledger sequences.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the `start-ledger` flag from registration in `AddArchiveFlags` / `AddCoreFlags` (default `2`) through `MustArchiveFlags` / `MustCoreFlags` parsing, into the input functions (`GetLedgers`, `GetTransactions`) that iterate `for seq := start; seq <= end; seq++`. Confirmed the default is indeed `2`, not `1`. Then examined the broader codebase design intent by reading `ledger_range.go`, `ValidateLedgerRange`, and `MustCoreFlags` error messages.

### Code Paths Examined

- `internal/utils/main.go:AddArchiveFlags:251` — registers `start-ledger` with default `2` and description "Defaults to genesis ledger"
- `internal/utils/main.go:AddCoreFlags:276` — identical registration for core-based exports
- `internal/utils/main.go:MustArchiveFlags:541-562` — reads and returns the flag value with no transformation
- `internal/utils/main.go:MustCoreFlags:596-628` — same, with error messages explicitly stating "genesis ledger (ledger 1)"
- `internal/utils/main.go:ValidateLedgerRange:736-743` — rejects ledger 0 but accepts ledger 1, confirming ledger 1 is a valid input
- `internal/input/ledger_range.go:55-57` — explains "Ledger sequence 2 is the start ledger because the genesis ledger (ledger 1), has a close time of 0 in Unix time"
- `internal/input/ledger_range.go:80` — "the second ledger has a real close time, unlike the 1970s close time of the genesis ledger"
- `internal/input/ledgers.go:GetLedgers:24` — `for seq := start; seq <= end; seq++` inclusive loop
- `internal/input/transactions.go:GetTransactions:37` — identical inclusive loop

### Why It Failed

This is **working-as-designed behavior**, not a data correctness bug. The codebase intentionally treats ledger 2 as the practical starting point because ledger 1 (the genesis ledger) is a special-case ledger with:

1. **Close time of Unix 0** (January 1, 1970) — not a real timestamp
2. **No transactions, operations, effects, or trades** — ledger 1 contains only the initial system state
3. **No meaningful export data** for any command except possibly `export_ledgers` (which would get a header with close time 0)

The design intent is consistent across the entire codebase: `ledger_range.go` starts searches at ledger 2 with explicit comments explaining why, `createNewGraph` sets `BeginPoint` to ledger 2, and all default flag values use 2. The flag description text "Defaults to genesis ledger" is misleading (it should say "Defaults to ledger 2" or "Defaults to first non-genesis ledger"), but this is a documentation wording issue, not a code bug that produces incorrect output.

The exported data for ledgers 2+ is fully correct. No data is corrupted, no fields are wrong. The only effect is that ledger 1 is excluded by default — which is intentional because it contains no actionable data.

### Lesson Learned

A mismatch between flag help text and flag default value is a documentation issue, not a data correctness bug. To qualify as a data-integrity finding, the behavior must produce incorrect output — not merely omit a special-case ledger that contains no meaningful data by design.
