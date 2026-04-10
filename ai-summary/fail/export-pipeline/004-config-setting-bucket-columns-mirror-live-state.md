# H002: Config-setting export fills `bucket_list_*` columns from live Soroban state fields

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Wrong field mappings in config-setting exports
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When exporting `config_settings`, only columns backed by the active XDR union arm should be populated. Bucket-list-specific output fields should stay zero or empty unless the active config-setting entry actually contains distinct bucket-list values.

## Mechanism

`TransformConfigSetting()` aliases several live Soroban state fields into unrelated bucket-list columns: `SorobanStateTargetSizeBytes` into `BucketListTargetSizeBytes`, rent-fee low/high into `WriteFee1KbBucketListLow/High`, `SorobanStateRentFeeGrowthFactor` into `BucketListWriteFeeGrowthFactor`, `LiveSorobanStateSizeWindowSampleSize` into `BucketListSizeWindowSampleSize`, and `GetLiveSorobanStateSizeWindow()` into `BucketListSizeWindow`. The exported JSON row therefore claims on-chain bucket-list settings that were never present in the source entry, and Parquet inherits the same wrong values because it is converted from that JSON struct.

## Trigger

Run `export_ledger_entry_changes --export-config-settings` on ledgers that contain non-zero `CONFIG_SETTING_CONTRACT_LEDGER_COST_V0`, `CONFIG_SETTING_STATE_ARCHIVAL`, or `CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW` entries.

## Target Code

- `internal/transform/config_setting.go:34-62` — bucket-list target/fee columns are sourced from `contractLedgerCost` Soroban-state fields
- `internal/transform/config_setting.go:85-107` — bucket-list window fields are sourced from `StateArchivalSettings.LiveSorobanStateSizeWindowSampleSize` and `GetLiveSorobanStateSizeWindow()`
- `internal/transform/config_setting.go:141-172` — aliased values are emitted into `BucketList*` JSON fields
- `internal/transform/schema.go:589-621` — bucket-list and live-state columns are distinct output fields

## Evidence

Current XDR exposes `SorobanStateTargetSizeBytes`, `RentFee1KbSorobanStateSizeLow/High`, and `SorobanStateRentFeeGrowthFactor` on `ConfigSettingContractLedgerCostV0`, and `LiveSorobanStateSizeWindowSampleSize` / `GetLiveSorobanStateSizeWindow()` on the state-archival and live-window config entries. There is no separate bucket-list source read anywhere in the transformer before those bucket-list columns are populated.

## Anti-Evidence

The schema still contains historical `bucket_list_*` column names, so some aliasing may have been intended for backward compatibility. But this repository's config-setting output is otherwise sparse by active union arm, so populating unrelated columns with copied live-state values breaks that pattern and produces data that looks authoritative even though it was not present on-chain.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

I traced the full git history of `config_setting.go` and `schema.go` across the Protocol 23 migration. Before Protocol 23, the XDR type `ConfigSettingContractLedgerCostV0` had fields named `BucketListTargetSizeBytes`, `WriteFee1KbBucketListLow`, `WriteFee1KbBucketListHigh`, and `BucketListWriteFeeGrowthFactor`. In Protocol 23, these were **renamed** (not replaced) to `SorobanStateTargetSizeBytes`, `RentFee1KbSorobanStateSizeLow`, `RentFee1KbSorobanStateSizeHigh`, and `SorobanStateRentFeeGrowthFactor`. The same pattern applies to `BucketListSizeWindowSampleSize` → `LiveSorobanStateSizeWindowSampleSize` and `GetBucketListSizeWindow()` → `GetLiveSorobanStateSizeWindow()`.

### Code Paths Examined

- `internal/transform/config_setting.go:34-62` — Both `bucketListTargetSizeBytes` and `sorobanStateTargetSizeBytes` are sourced from the same `contractLedgerCost.SorobanStateTargetSizeBytes`. Same pattern for all other aliased pairs.
- `internal/transform/config_setting.go:85-107` — `bucketListSizeWindowSampleSize` and `liveSorobanStateSizeWindowSampleSize` both come from `stateArchivalSettings.LiveSorobanStateSizeWindowSampleSize`. `bucketListSizeWindow` and `liveSorobanStateSizeWindow` both come from `GetLiveSorobanStateSizeWindow()`.
- `internal/transform/schema.go:589-621` — Both old `BucketList*` and new `Soroban*` columns exist as separate JSON fields in `ConfigSettingOutput`.
- Git commit `2c9ee7c` (Protocol 23 #340) — Introduced the dual-column pattern by keeping old `BucketList*` columns and adding new `Soroban*` columns, all sourced from the renamed XDR fields. The same backward-compatibility aliasing pattern was applied systematically to ALL renamed fields: `LedgerMaxReadLedgerEntries`/`LedgerMaxDiskReadEntries`, `FeeReadLedgerEntry`/`FeeDiskReadLedgerEntry`, `FeeRead1Kb`/`FeeDiskRead1Kb`.
- Git commit `c594c3a` (on a different branch) — Shows a version that removed old columns entirely, confirming the master branch's dual-column approach was a deliberate backward-compatibility choice.

### Why It Failed

The hypothesis describes **working-as-designed behavior**. The `BucketList*` columns are not "unrelated" to the `Soroban*` values — they are legacy names for the **exact same conceptual values** that were renamed in Protocol 23. The XDR renamed `BucketListTargetSizeBytes` to `SorobanStateTargetSizeBytes` (and similarly for all other fields in this group). The developer intentionally kept the old column names populated with the same values to maintain backward compatibility for downstream BigQuery consumers, while simultaneously adding new columns under the updated names. This is the same pattern applied to every other renamed field in Protocol 23 (e.g., `LedgerMaxReadLedgerEntries` → `LedgerMaxDiskReadEntries`). The data in the `BucketList*` columns is correct — it IS the same on-chain value, just under its old name.

### Lesson Learned

When XDR field renames occur during protocol upgrades, dual-column aliasing (old name + new name, same source) is an intentional backward-compatibility pattern, not a field-mapping bug. Check git history across protocol boundary commits before concluding that duplicated columns indicate wrong mappings.
