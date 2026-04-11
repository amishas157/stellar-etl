# H001: Fractional start times can select a ledger that closed before the requested window

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `get_ledger_range_from_times` receives a `--start-time` that falls **after** ledger `N` closed but **before** ledger `N+1` closed, the returned `start` ledger should be `N+1`. Downstream exports seeded from that range should therefore exclude ledger `N`, because its `closed_at` is earlier than the requested start instant.

## Mechanism

The command accepts timestamps with fractional seconds, and `time.Parse` preserves that sub-second precision in the resulting `time.Time`. `GetLedgerRange()` then compares candidate ledgers via `CloseTime.Unix()`, which truncates both the target and the ledger close times to whole seconds. If the requested start time is, for example, `T+0.100s` and ledger `N` closed at `T+0.000s`, the comparison treats them as equal-second values, fails the "previous < target" boundary check, and the recursive search can walk back to ledger `N` instead of advancing to `N+1`.

## Trigger

1. Pick two consecutive ledgers whose close times are `T` and `T+Δ`, with `Δ > 0`.
2. Run `stellar-etl get_ledger_range_from_times --start-time <T plus fractional milliseconds> --end-time <any later valid time>`.
3. The correct JSON should begin at ledger `N+1`, because ledger `N` closed before the requested start instant.
4. The current path can return ledger `N`, causing every export that uses this range to include at least one pre-window ledger.

## Target Code

- `cmd/get_ledger_range_from_times.go:21-25` — public contract advertises `.SSS` timestamps in the command help text
- `cmd/get_ledger_range_from_times.go:52-63` — parses the full-precision timestamp and passes it into `GetLedgerRange`
- `internal/input/ledger_range.go:GetLedgerRange:31-67` — uses the same search helper for the returned `start` boundary
- `internal/input/ledger_range.go:findLedgerForTimeBinary:117-122` — binary-search boundary test truncates times with `Unix()`
- `internal/input/ledger_range.go:findLedgerForDate:158-170` — heuristic search truncates to whole seconds, then steps backward when the target is slightly earlier than the current point

## Evidence

The command help explicitly documents millisecond-bearing input, and a standalone Go parse of the documented format preserves fractional nanoseconds. The search path does not compare `time.Time` values directly anywhere; every key boundary test reduces them to `Unix()` seconds first. That creates a concrete mismatch where accepted CLI precision is silently discarded before the range boundary is chosen.

## Anti-Evidence

Stellar ledger close times are second-granularity values, so whole-second inputs do not hit this path. The hypothesis therefore depends on callers actually using the documented fractional-second format rather than integer-second timestamps only.
