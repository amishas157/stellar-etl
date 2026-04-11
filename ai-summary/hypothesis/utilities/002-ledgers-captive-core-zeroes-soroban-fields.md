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
