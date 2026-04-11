# H001: `export_ledgers --captive-core` drops LCM-derived Soroban ledger metadata

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_ledgers --captive-core` should emit the same Soroban-era ledger metadata as the default backend for the same ledger range. In particular, `soroban_fee_write_1kb`, `total_byte_size_of_live_soroban_state`, and `evicted_ledger_keys_{type,hash}` should reflect the actual `LedgerCloseMeta` contents instead of falling back to zero values.

## Mechanism

When `--captive-core` is set, `cmd/export_ledgers.go` switches away from `GetLedgers()` and into `GetLedgersHistoryArchive()`. That history-archive reader only fills `HistoryArchiveLedgerAndLCM.Ledger` and leaves `LCM` as the zero-value union, but `TransformLedger()` populates the Soroban-only columns exclusively from `lcm.GetV1()` / `lcm.GetV2()`. The exported row still looks otherwise valid because header-level fields come from `historyarchive.Ledger`, so the missing Soroban metadata is silent corruption rather than a crash.

## Trigger

Run `stellar-etl export_ledgers --captive-core --start-ledger <L> --end-ledger <R>` over a Soroban-era range where at least one ledger has non-zero `SorobanFeeWrite1Kb`, non-zero `TotalByteSizeOfLiveSorobanState`, or non-empty `EvictedKeys`. Compare the output against the same range without `--captive-core`: the captive-core run should preserve those values, but this path will emit zeros / empty arrays.

## Target Code

- `cmd/export_ledgers.go:28-43` — `--captive-core` selects `GetLedgersHistoryArchive()` before calling `TransformLedger`
- `internal/input/ledgers_history_archive.go:GetLedgersHistoryArchive:10-28` — returns `HistoryArchiveLedgerAndLCM{Ledger: ledger}` with no `LCM`
- `internal/transform/ledger.go:61-90` — Soroban ledger fields are populated only from `lcm.GetV1()` / `lcm.GetV2()`
- `internal/transform/ledger.go:104-129` — zero-valued Soroban fields are written into the exported row

## Evidence

`GetLedgersHistoryArchive()` never assigns the `LCM` field at all, while `TransformLedger()` has no alternate source for the Soroban-only columns. `internal/transform/ledger_test.go:101-109,192-236` also shows those columns are expected to come from the supplied `LedgerCloseMeta`, not from the history-archive header alone.

## Anti-Evidence

Pre-Soroban ledgers, or Soroban ledgers where those values are genuinely zero / empty, will mask the bug because the incorrect zero-value row looks plausible. The corruption is limited to the `--captive-core` branch of `export_ledgers`; the default datastore-backed path still passes a real `LedgerCloseMeta`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published`
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same bug already confirmed and published as success finding 001 in the data-input subsystem. Both identify `export_ledgers --captive-core` routing to `GetLedgersHistoryArchive()` which omits `LCM`, causing `TransformLedger()` to emit zero-valued Soroban metadata fields. The mechanism, target code paths, and severity are identical.

### Code Paths Examined

- `cmd/export_ledgers.go:28-43` — confirmed same `UseCaptiveCore` → `GetLedgersHistoryArchive()` routing described in success/001
- `internal/input/ledgers_history_archive.go:10-28` — confirmed same missing `LCM` field described in success/001
- `internal/transform/ledger.go:61-90` — confirmed same Soroban field population from `lcm.GetV1()`/`lcm.GetV2()` described in success/001

### Why It Failed

This is a direct duplicate of the already-confirmed and published finding `ai-summary/success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published`. The root cause, mechanism, affected code paths, severity, and suggested fix are all substantively identical.

### Lesson Learned

Before generating hypotheses about `--captive-core` flag routing in `export_ledgers`, check whether the captive-core/history-archive LCM population gap has already been documented.
