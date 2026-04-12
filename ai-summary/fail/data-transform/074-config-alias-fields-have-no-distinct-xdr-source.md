# H074: Config-setting alias columns looked mis-mapped, but the protocol exposes no distinct source fields

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `config_settings` exports distinct metrics such as `ledger_max_read_ledger_entries` versus `ledger_max_disk_read_entries`, or `bucket_list_target_size_bytes` versus `soroban_state_target_size_bytes`, each column should be populated from a distinct XDR field that can differ on-chain. A decoded config-setting row should not mirror one protocol value into two semantically different output columns unless the protocol explicitly defines them as aliases.

## Mechanism

`TransformConfigSetting()` initially looks like a copy-paste bug because it assigns multiple schema fields from the same XDR members: disk-read values fill both `read` and `disk_read` columns, and Soroban-state rent fields fill both `bucket_list_*` and `soroban_state_*` columns. That would be a data-corruption bug if the union arm exposed distinct source members for the duplicated outputs.

## Trigger

1. Inspect `TransformConfigSetting()` for `CONFIG_SETTING_CONTRACT_LEDGER_COST_V0`, `CONFIG_SETTING_STATE_ARCHIVAL`, and `CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW`.
2. Compare the accessed XDR members against the current SDK definitions for those config-setting structs.

## Target Code

- `internal/transform/config_setting.go:34-61` — duplicates disk-read and Soroban-state values into multiple output fields
- `internal/transform/config_setting.go:85-107` — duplicates live-state archival window values into both `bucket_list_*` and `live_soroban_state_*` outputs

## Evidence

The transform explicitly assigns `contractLedgerCost.FeeDiskReadLedgerEntry` to both `FeeReadLedgerEntry` and `FeeDiskReadLedgerEntry`, and `contractLedgerCost.SorobanStateTargetSizeBytes` to both `BucketListTargetSizeBytes` and `SorobanStateTargetSizeBytes`. It does the same for `LiveSorobanStateSizeWindowSampleSize` and the live-state window slice itself.

## Anti-Evidence

The current XDR definitions only expose disk-read members in `ConfigSettingContractLedgerCostV0` and only live-Soroban-state members in `StateArchivalSettings` / `ConfigSettingEntry`. There is no distinct generic-read or bucket-list member to map instead.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The apparent duplicate mappings are driven by schema aliases, not by the transform selecting the wrong XDR field. The protocol simply does not define separate source members for the alternative column names, so this is not a live transform bug with a correct unused source path.

### Lesson Learned

For config-setting rows, duplicated-looking output columns are only actionable when the underlying XDR struct exposes distinct members and the transform chooses the wrong one. If the protocol has only one source field, the issue is a schema-contract question, not transform-layer corruption.
