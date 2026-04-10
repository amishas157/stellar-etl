# H004: Bucket-list config columns are populated with live-state window values

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW` and `CONFIG_SETTING_STATE_ARCHIVAL` rows, the `live_soroban_state_*` columns should reflect the active XDR fields, while `bucket_list_*` columns should stay empty/default unless there is a distinct bucket-list source. A live-state window like `[100,200]` should not also appear in `bucket_list_size_window`.

## Mechanism

`TransformConfigSetting()` only reads `GetLiveSorobanStateSizeWindow()` and `StateArchivalSettings.LiveSorobanStateSizeWindowSampleSize`, then copies those live-state values into both the `BucketList*` and `LiveSorobanState*` fields. The Parquet converter preserves the duplicated arrays, so downstream analytics see two differently named columns with the same payload.

## Trigger

Export any config-setting row with `ConfigSettingIdConfigSettingLiveSorobanStateSizeWindow` or `ConfigSettingIdConfigSettingStateArchival` and a non-empty live-state window or non-zero live-state sample size.

## Target Code

- `internal/transform/config_setting.go:92-107` — derives both sample-size fields and both window slices from live-state inputs
- `internal/transform/config_setting.go:165-172` — assigns the duplicated values into the output struct
- `internal/transform/schema.go:613-621` — exposes separate bucket-list and live-state columns
- `internal/transform/parquet_converter.go:339-345` — builds only one converted slice
- `internal/transform/parquet_converter.go:397-405` — writes the same slice into both Parquet columns

## Evidence

There is a getter for `GetLiveSorobanStateSizeWindow()`, and `StateArchivalSettings` exposes `LiveSorobanStateSizeWindowSampleSize`, but the transformer assigns those values to `BucketListSizeWindow`, `BucketListSizeWindowSampleSize`, `LiveSorobanStateSizeWindow`, and `LiveSorobanStateSizeWindowSampleSize`. No separate bucket-list getter is consulted anywhere in this path.

## Anti-Evidence

If downstream code intentionally treats the bucket-list columns as aliases, the duplication may not break that consumer. The schema names, however, imply distinct metrics, so silently backfilling bucket-list columns with live-state values is misleading.
