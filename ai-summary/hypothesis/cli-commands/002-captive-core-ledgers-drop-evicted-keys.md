# H002: `export_ledgers --captive-core` drops evicted-ledger-key arrays

**Date**: 2026-04-14
**Subsystem**: cli-commands
**Severity**: High
**Impact**: missing eviction metadata in ledger exports
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a ledger's `LedgerCloseMeta` contains `EvictedKeys`, then `export_ledgers` should emit matching `evicted_ledger_keys_type` and `evicted_ledger_keys_hash` arrays regardless of whether the operator requested the default datastore reader or `--captive-core`.

## Mechanism

The `--captive-core` branch in `export_ledgers` bypasses the ledger-backend path and reads plain history-archive ledgers, which carry no `LedgerCloseMeta` payload in `HistoryArchiveLedgerAndLCM.LCM`. `TransformLedger()` only derives `evicted_ledger_keys_type/hash` from `lcm.GetV1()` / `lcm.GetV2()`, so ledgers that really evicted keys are exported with empty arrays on the `--captive-core` path.

## Trigger

Run `export_ledgers --captive-core` over a ledger range containing at least one ledger with non-empty `EvictedKeys` in close meta. The exported row for that ledger will show empty/null `evicted_ledger_keys_type` and `evicted_ledger_keys_hash`, while the same ledger exported through the normal backend path can populate those arrays.

## Target Code

- `cmd/export_ledgers.go:28-32` — routes `--captive-core` through the history-archive helper
- `internal/input/ledgers_history_archive.go:GetLedgersHistoryArchive:10-34` — leaves `LCM` unset for every returned ledger
- `internal/transform/ledger.go:63-90` — only populates eviction metadata from `LedgerCloseMeta` V1/V2
- `internal/transform/ledger.go:125-128` — writes the empty `evicted_ledger_keys_*` slices into the final `LedgerOutput`

## Evidence

`TransformLedger()` initializes `outputEvictedKeysHash` and `outputEvictedKeysType` as nil slices and only calls `transformLedgerKeys()` inside the `lcm.GetV1()` / `lcm.GetV2()` branches. Because the history-archive path never populates `LCM`, those branches never run under `--captive-core`, even though real close meta can carry `EvictedKeys`.

## Anti-Evidence

Many ledgers do not evict any keys, so empty arrays are sometimes correct. The bug only appears on ledgers where eviction metadata is present in close meta, but on those ledgers the wrong output is still success-shaped and plausible.
