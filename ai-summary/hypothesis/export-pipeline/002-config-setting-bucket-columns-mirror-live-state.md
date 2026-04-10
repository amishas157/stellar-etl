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
