# H022: Config-setting Parquet live-state window aliases bucket-list window

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `bucket_list_size_window` and `live_soroban_state_size_window` ever differ in a `ConfigSettingOutput`, the Parquet converter should preserve each slice independently. A populated `live_soroban_state_size_window` should not be overwritten by the bucket-list slice during conversion.

## Mechanism

`ConfigSettingOutput.ToParquet()` builds one converted slice named `BucketListSizeWindowInt`, then assigns that same slice to both `BucketListSizeWindow` and `LiveSorobanStateSizeWindow`. On its face, that looks like a copy-paste bug that would make the Parquet live-state window mirror the bucket-list window instead of the JSON value.

## Trigger

Export a config-setting row where `ConfigSettingOutput.BucketListSizeWindow` and `ConfigSettingOutput.LiveSorobanStateSizeWindow` contain different values, then convert it to Parquet.

## Target Code

- `internal/transform/parquet_converter.go:ConfigSettingOutput.ToParquet:339-410` — `LiveSorobanStateSizeWindow` is assigned from `BucketListSizeWindowInt`
- `internal/transform/config_setting.go:TransformConfigSetting:101-107,171-172` — transform populates both JSON slices from the same `GetLiveSorobanStateSizeWindow()` data
- `internal/transform/config_setting_test.go:141-148` — checked-in expectations assert both slices are identical

## Evidence

The Parquet converter contains a literal suspicious assignment: `LiveSorobanStateSizeWindow: BucketListSizeWindowInt`. If the source struct held two different windows, that line would definitely collapse them.

## Anti-Evidence

The current transform never produces different source slices. `TransformConfigSetting()` populates both fields from the same `GetLiveSorobanStateSizeWindow()` call, and the repository test fixture explicitly expects both fields to equal the same `bucket` slice. Because the JSON-layer values are already identical, the Parquet aliasing typo does not change observable output today.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspicious Parquet assignment is real, but it is dead code in the current implementation because `TransformConfigSetting()` already duplicates the same source slice into both fields. There is no present-day input that makes the Parquet row differ from the JSON row on this path.

### Lesson Learned

A converter typo is only a live data-integrity bug if the pre-conversion fields can actually diverge in current code. For config-setting investigations, check the JSON-layer transform and checked-in fixtures before treating a bad Parquet assignment as observable corruption.
