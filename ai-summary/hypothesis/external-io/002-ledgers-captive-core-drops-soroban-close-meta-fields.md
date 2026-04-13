# H002: `export_ledgers --captive-core` silently zeroes Soroban close-meta fields

**Date**: 2026-04-13
**Subsystem**: external-io
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledgers --captive-core` should export the same per-ledger Soroban metadata as the default ledger path for the same ledger range, including `soroban_fee_write_1kb`, `total_byte_size_of_bucket_list`, `total_byte_size_of_live_soroban_state`, and any `evicted_ledger_keys_*` values present in the ledger close meta.

## Mechanism

The `--captive-core` branch switches to `GetLedgersHistoryArchive()`, which returns only `historyarchive.Ledger` and leaves `HistoryArchiveLedgerAndLCM.LCM` unset. `TransformLedger()` fills the Soroban-specific output columns only when `lcm.GetV1()` or `lcm.GetV2()` succeeds, so this branch silently exports zeros and empty arrays for any ledger whose real close meta carries non-default Soroban values.

## Trigger

Run `stellar-etl export_ledgers --captive-core --start-ledger <s> --end-ledger <e>` over Soroban-era ledgers where the real `LedgerCloseMeta` has non-zero `SorobanFeeWrite1Kb`, non-zero live-state size, or non-empty evicted keys.

## Target Code

- `cmd/export_ledgers.go:28-43` — routes `--captive-core` to `GetLedgersHistoryArchive()` and passes the returned `ledger.LCM` into `TransformLedger()`
- `internal/input/ledgers_history_archive.go:16-29` — appends `HistoryArchiveLedgerAndLCM{Ledger: ledger}` without populating `LCM`
- `internal/transform/ledger.go:61-90` — populates Soroban ledger fields only from `lcm.GetV1()` / `lcm.GetV2()`
- `internal/transform/schema.go:31-37` — exposes the affected JSON columns in `LedgerOutput`

## Evidence

The default `GetLedgers()` path explicitly stores both `Ledger` and `LCM`, but the history-archive path only stores `Ledger`. Inside `TransformLedger()`, every affected field is gated behind `lcm.GetV1()`/`GetV2()`, so a zero-value `LCM` means the export falls through to the zero-value defaults even though the selected ledger may have real Soroban metadata.

## Anti-Evidence

Classic pre-Soroban ledgers legitimately have zero/empty values for these columns, so the trigger must target ledgers where the real close meta is known to populate them. Core ledger fields such as sequence, hashes, and counts still come from `historyarchive.Ledger` and should remain correct.
