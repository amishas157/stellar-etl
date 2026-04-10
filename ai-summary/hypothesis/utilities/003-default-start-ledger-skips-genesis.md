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

The flag builders use `flags.Uint32P("start-ledger", "s", 2, ...)` while the help text says “Defaults to genesis ledger,” and multiple command paths call `MustArchiveFlags`/`MustCoreFlags` directly before handing the result to inclusive `for seq := start; seq <= end; seq++` loops. The README repeats the same contradiction by documenting “Defaults to genesis ledger” alongside a default value of `2`.

## Anti-Evidence

`internal/input/ledger_range.go` intentionally starts time-to-ledger searches at ledger `2` because ledger `1` has Unix close time `0`. That rationale is specific to date-range conversion, though, and it does not justify silently excluding ledger `1` from normal export commands that work on explicit ledger sequences.
