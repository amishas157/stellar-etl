# H002: `export_ledgers --captive-core` zeroes LCM-only Soroban ledger fields

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Protocol 23+ ledgers that carry Soroban metadata, `export_ledgers --captive-core` should emit the real values for `soroban_fee_write_1kb`, `total_byte_size_of_bucket_list`, `total_byte_size_of_live_soroban_state`, `evicted_ledger_keys_hash`, and `evicted_ledger_keys_type`. Those values live in `LedgerCloseMeta.V1/V2`, so the export should preserve them whenever the source ledger contains them.

## Mechanism

When `--captive-core` is set, `export_ledgers` routes to `GetLedgersHistoryArchive()`, which constructs `utils.HistoryArchiveLedgerAndLCM` with only the history-archive `Ledger` populated and leaves `LCM` as the zero value. `TransformLedger()` only fills the Soroban-only output columns from `lcm.GetV1()` / `lcm.GetV2()`; with an empty `LCM`, both checks fail and the function silently exports the zero defaults instead of the true ledger metadata.

## Trigger

Run `stellar-etl export_ledgers --captive-core -s <start> -e <end>` across any legitimate Protocol 23+ ledger where `LedgerCloseMeta.V1/V2` contains non-zero Soroban metadata or evicted keys. Inspect the exported JSON / Parquet row for that ledger and compare it to the same ledger exported through the normal `GetLedgers()` path.

## Target Code

- `cmd/export_ledgers.go:28-31` - selects `GetLedgersHistoryArchive(...)` when `UseCaptiveCore` is true
- `internal/input/ledgers_history_archive.go:24-28` - populates `HistoryArchiveLedgerAndLCM{Ledger: ledger}` without setting `LCM`
- `internal/transform/ledger.go:61-91` - initializes Soroban-only outputs to zero values and only overwrites them from `lcm.GetV1()` / `lcm.GetV2()`
- `internal/input/ledgers.go:79-82` - normal path shows the intended contract: both `Ledger` and `LCM` are populated

## Evidence

The history-archive helper never fills the `LCM` half of the transport struct, yet `TransformLedger()` relies on `LCM` for every Soroban-only field. Because those output variables are initialized to `0`, `nil`, or empty slices before the `GetV1()` / `GetV2()` checks, the export produces plausible-but-wrong zero values instead of failing.

## Anti-Evidence

This only corrupts ledgers whose correct values are non-zero or non-empty; pre-Soroban ledgers legitimately export zeros there. The default non-`--captive-core` path also avoids the issue because `GetLedgers()` carries the real `LCM`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of ai-summary/success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published
**Failed At**: reviewer

### Trace Summary

Traced the code path from `cmd/export_ledgers.go:28` where `UseCaptiveCore=true` routes to `GetLedgersHistoryArchive()`, through `internal/input/ledgers_history_archive.go:24-26` where `HistoryArchiveLedgerAndLCM{Ledger: ledger}` is constructed without setting `LCM`, to `internal/transform/ledger.go:61-91` where `lcm.GetV1()` and `lcm.GetV2()` both return `ok=false` on the zero-valued LCM, leaving all Soroban output fields at their zero defaults. The mechanism described in the hypothesis is correct.

### Code Paths Examined

- `cmd/export_ledgers.go:28-31` — confirms `UseCaptiveCore` routes to `GetLedgersHistoryArchive()`
- `internal/input/ledgers_history_archive.go:24-26` — confirms `LCM` field is never populated
- `internal/transform/ledger.go:61-91` — confirms Soroban fields come exclusively from `lcm.GetV1()` / `lcm.GetV2()`
- `internal/input/ledgers.go:79-82` — confirms the normal path populates both `Ledger` and `LCM`

### Why It Failed

This is an exact duplicate of an already-confirmed and published finding. The bug was previously investigated under the `data-input` subsystem and confirmed as a High-severity success in `ai-summary/success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published`. That investigation covers the identical code paths (`cmd/export_ledgers.go` → `GetLedgersHistoryArchive()` → zero-valued `LCM` → `TransformLedger` zeroes Soroban fields), the identical affected fields (`SorobanFeeWrite1Kb`, `TotalByteSizeOfLiveSorobanState`, evicted keys), and includes a complete PoC test. There is also an existing PoC in `ai-summary/poc/utilities/001-captive-core-flag-routes-to-history-archive.md` covering this same finding.

### Lesson Learned

Cross-subsystem duplicate checking is essential. This hypothesis was filed under `utilities` but the identical bug was already confirmed under `data-input`. The bug spans both subsystems (the input layer omits LCM, the transform layer silently accepts the zero value), so hypotheses about it can originate from either subsystem's perspective.
