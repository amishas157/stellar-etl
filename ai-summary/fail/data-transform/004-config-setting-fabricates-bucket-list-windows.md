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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantially equivalent to fail/data-transform/003-ledger-bucket-list-size-mirrors-live-state.md (same root cause, different function)
**Failed At**: reviewer

### Trace Summary

Traced `TransformConfigSetting()` in `internal/transform/config_setting.go` lines 85–107 and 159–172. Confirmed that `BucketListSizeWindowSampleSize` and `BucketListSizeWindow` are sourced from `stateArchivalSettings.LiveSorobanStateSizeWindowSampleSize` and `configSetting.GetLiveSorobanStateSizeWindow()` respectively — the same sources as the `LiveSorobanState*` counterparts. Then inspected the upstream XDR definition in `Stellar-contract-config-setting.x`. The `StateArchivalSettings` struct defines only `liveSorobanStateSizeWindowSampleSize` and `liveSorobanStateSizeWindowSamplePeriod` — there are no `bucketListSizeWindowSampleSize` or `bucketListSizeWindow` fields anywhere in the XDR. The `ConfigSettingEntry` union's `CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW` arm provides `uint64 liveSorobanStateSizeWindow<>` with no bucket-list counterpart.

### Code Paths Examined

- `Stellar-contract-config-setting.x:305-325` — `StateArchivalSettings` XDR struct defines only `liveSorobanStateSizeWindowSampleSize` and `liveSorobanStateSizeWindowSamplePeriod`; no bucket-list fields exist
- `Stellar-contract-config-setting.x:397` — `CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW` arm provides only `uint64 liveSorobanStateSizeWindow<>`; no bucket-list window arm exists
- `internal/transform/config_setting.go:92-93` — both `bucketListSizeWindowSampleSize` and `liveSorobanStateSizeWindowSampleSize` read from `stateArchivalSettings.LiveSorobanStateSizeWindowSampleSize` (the only available source)
- `internal/transform/config_setting.go:101-107` — both slices built from `configSetting.GetLiveSorobanStateSizeWindow()` (the only available source)
- `internal/transform/parquet_converter.go:341-345,403-404` — `BucketListSizeWindowInt` reused for both Parquet columns; functionally identical since source values are the same

### Why It Failed

The hypothesis assumes the Stellar XDR protocol exposes separate bucket-list-window and live-Soroban-state-window fields, and that the ETL erroneously reads only one. In reality, the Stellar protocol defines only live-Soroban-state fields in both `StateArchivalSettings` (sample size/period) and the `ConfigSettingEntry` union (window array). There are no bucket-list-window counterparts anywhere in the XDR. The ETL's bucket-list-named columns are backward-compatible aliases retained from before the protocol renamed these fields — they correctly contain the same values as the live-Soroban-state columns because there is only one data source. The Parquet converter sharing the same `BucketListSizeWindowInt` slice for both columns is a minor code quality shortcut that produces identical output. This is the exact same root cause pattern investigated and rejected in `003-ledger-bucket-list-size-mirrors-live-state.md`, applied to a different function.

### Lesson Learned

The "bucket list" → "live Soroban state" renaming was a protocol-wide change. All ETL columns named `bucket_list_*` that map to `liveSorobanState*` XDR sources follow the same backward-compatible alias pattern. Future hypotheses about bucket-list/live-state column duplication should first verify whether distinct XDR source fields exist, regardless of which transform function is involved.
