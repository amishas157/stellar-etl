# H007: Config-setting parquet reuses `bucket_list_size_window` for `live_soroban_state_size_window`

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `ConfigSettingOutput` can carry distinct `bucket_list_size_window` and `live_soroban_state_size_window` arrays, `ToParquet()` should serialize each array independently so parquet preserves the same values as JSON.

## Mechanism

`ConfigSettingOutput.ToParquet()` converts only `BucketListSizeWindow` into `BucketListSizeWindowInt` and then assigns that same converted slice to both `BucketListSizeWindow` and `LiveSorobanStateSizeWindow` in the parquet struct. At first glance, that looks like a copy-paste mapping bug that could make parquet claim the live-state window equals the bucket-list window even when JSON differs.

## Trigger

Inspect the config-setting parquet conversion path for rows that would hypothetically populate different `bucket_list_size_window` and `live_soroban_state_size_window` arrays.

## Target Code

- `internal/transform/parquet_converter.go:ConfigSettingOutput.ToParquet:339-410` — converts only one source slice and assigns it to both parquet fields
- `internal/transform/config_setting.go:TransformConfigSetting:101-107,171-172` — builds both JSON arrays from the same source slice
- `internal/transform/config_setting_test.go:makeConfigSettingTestOutput:141-148` — expected outputs already model both arrays as identical

## Evidence

The parquet converter clearly reuses `BucketListSizeWindowInt` for both fields at lines 403-404. If the JSON struct ever diverged, the parquet path would flatten that distinction.

## Anti-Evidence

The current JSON transform does not produce distinct arrays in the first place. `TransformConfigSetting()` populates both arrays from `configSetting.GetLiveSorobanStateSizeWindow()`, and the current tests expect both fields to share the same `bucket` slice. So the parquet reuse does not create a new wrong value on today's reachable export path.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The parquet converter is suspicious, but the live JSON source already aliases both arrays to the same data. Because the upstream transform never produces distinct values here, reusing the converted bucket-list slice in parquet does not currently change any exported value.

### Lesson Learned

Parquet copy-paste bugs only become data-integrity findings when the upstream JSON struct can actually carry different values for the affected fields. If the source transform already aliases the columns, the parquet bug is dormant until that upstream behavior changes.
