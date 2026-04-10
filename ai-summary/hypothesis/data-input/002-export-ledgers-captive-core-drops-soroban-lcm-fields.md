# H002: export_ledgers Captive-Core Path Drops Soroban LCM Fields

**Date**: 2026-04-10
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_ledgers` should populate Soroban-derived ledger fields such as `soroban_fee_write_1kb`, `total_byte_size_of_live_soroban_state`, `evicted_ledger_keys_type`, and `evicted_ledger_keys_hash` from the ledger's close meta whenever those values exist on-chain, regardless of backend selection.

## Mechanism

`cmd/export_ledgers.go` also inverts the `UseCaptiveCore` branch and calls `GetLedgersHistoryArchive()` when captive core is requested. That reader returns `HistoryArchiveLedgerAndLCM` values with only `Ledger` populated; `LCM` remains zero-valued, and `TransformLedger()` only fills the Soroban-specific columns by reading `lcm.GetV1()` / `lcm.GetV2()`, so those columns silently become `0` or empty slices even for ledgers whose real close meta contains non-zero values.

## Trigger

Run `stellar-etl export_ledgers --use-captive-core --start-ledger <L> --end-ledger <R>` over a Soroban-heavy range where the real `LedgerCloseMeta` includes `SorobanFeeWrite1Kb`, live-state byte size, or evicted keys. Compare the JSON/Parquet output with the same range exported without `--use-captive-core` or with the raw close meta.

## Target Code

- `cmd/export_ledgers.go:28-43` — selects `GetLedgersHistoryArchive()` for captive-core exports and passes the returned `LCM` into `TransformLedger`
- `internal/input/ledgers_history_archive.go:10-28` — constructs `HistoryArchiveLedgerAndLCM` without setting the `LCM` field
- `internal/utils/main.go:1122-1125` — defines the wrapper struct whose zero-value `LCM` flows downstream
- `internal/transform/ledger.go:61-91` — populates Soroban ledger fields only from `lcm.GetV1()` / `lcm.GetV2()`

## Evidence

The history-archive reader intentionally omits `LCM`, while the normal `GetLedgers()` path preserves it from `backend.GetLedger()`. `TransformLedger()` gets most basic ledger fields from `historyarchive.Ledger`, but every Soroban-specific field is gated on a non-empty `LedgerCloseMeta`, so the branch inversion creates a partial row that looks valid except the Soroban columns are zeroed out.

## Anti-Evidence

Core header-derived fields like sequence, close time, transaction counts, and hashes still come from `historyarchive.Ledger`, so the exported row is not obviously broken. The corruption is concentrated in the close-meta-derived Soroban columns.
