# H003: Config setting bucket-list window fields are fabricated from live Soroban state

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `CONFIG_SETTING_STATE_ARCHIVAL` and `CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW` rows, the exporter should only populate columns that are backed by those XDR fields. Bucket-list window columns should stay zero/empty unless there is a distinct bucket-list source, while the live-Soroban-state columns should preserve the actual XDR values.

## Mechanism

`TransformConfigSetting()` copies `stateArchivalSettings.LiveSorobanStateSizeWindowSampleSize` into both `BucketListSizeWindowSampleSize` and `LiveSorobanStateSizeWindowSampleSize`, then calls `GetLiveSorobanStateSizeWindow()` once and appends the same slice into both `BucketListSizeWindow` and `LiveSorobanStateSizeWindow`. The Parquet converter compounds this by converting only `BucketListSizeWindow` and reusing that converted slice for `live_soroban_state_size_window` as well.

## Trigger

Export config settings that include non-zero `CONFIG_SETTING_STATE_ARCHIVAL` or `CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW` values. The JSON row will show bucket-list window fields matching live-Soroban-state fields, and the Parquet row will carry the same duplication.

## Target Code

- `internal/transform/config_setting.go:TransformConfigSetting:85-107` — sources bucket-list sample size and window data from live-Soroban-state fields
- `internal/transform/config_setting.go:TransformConfigSetting:159-172` — writes those duplicated values into both JSON columns
- `internal/transform/parquet_converter.go:ConfigSettingOutput.ToParquet:339-410` — reuses the bucket-list slice for `LiveSorobanStateSizeWindow` in Parquet
- `internal/transform/schema.go:ConfigSettingOutput:563-621` — schema exposes distinct bucket-list and live-state fields

## Evidence

The current `xdr.ConfigSettingEntry` union exposes `GetLiveSorobanStateSizeWindow()` and `StateArchivalSettings.LiveSorobanStateSizeWindowSampleSize`, but no bucket-list-window counterpart. Despite that, the transformer fills both bucket-list columns from those live-state sources at `config_setting.go:92-106`, and `parquet_converter.go:403-404` writes both repeated Parquet columns from the single `BucketListSizeWindowInt` slice.

## Anti-Evidence

The duplicated bucket-list columns may exist as backward-compatible aliases for an older schema. But if that is intentional, the ETL is still exporting synthetic bucket-list measurements that look authoritative and are indistinguishable from genuinely sourced values.
