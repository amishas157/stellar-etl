# H004: Config-setting Parquet window duplication is not distinct from the current JSON export

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `ConfigSettingOutput.ToParquet()` is corrupting `live_soroban_state_size_window`, then a config-setting row whose JSON output has distinct `bucket_list_size_window` and `live_soroban_state_size_window` values should preserve that distinction in Parquet as well.

## Mechanism

I investigated whether `ToParquet()` wrongly copies `BucketListSizeWindowInt` into both repeated Parquet fields, which would make `live_soroban_state_size_window` mirror the bucket-list window even when the JSON row differs. That would be a concrete Parquet-only corruption if the upstream transform preserved distinct arrays.

## Trigger

Export config settings with `export_ledger_entry_changes --export-config-settings --write-parquet` on a ledger whose config-setting row would need different values for `bucket_list_size_window` and `live_soroban_state_size_window`.

## Target Code

- `internal/transform/parquet_converter.go:339-405` - Parquet converter reuses `BucketListSizeWindowInt` for both repeated fields
- `internal/transform/config_setting.go:101-107` - JSON transform already populates both slices from the same getter
- `internal/transform/config_setting.go:171-172` - emitted JSON struct stores identical arrays in both fields
- `internal/transform/config_setting_test.go:147-148` - current tests expect both fields to contain the same `bucket` slice

## Evidence

`ConfigSettingOutput.ToParquet()` clearly assigns `BucketListSizeWindowInt` to both `BucketListSizeWindow` and `LiveSorobanStateSizeWindow`. On its face, that looks like a copy-paste mapping bug.

## Anti-Evidence

The earlier transform layer already fills both JSON fields from the same `GetLiveSorobanStateSizeWindow()` source and the current tests assert that both arrays are identical. In the current codebase, Parquet is not introducing a new divergence from JSON on this path.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This repository already emits identical values for these two config-setting fields before Parquet conversion happens, so the suspicious Parquet assignment is not a distinct external-io corruption path by itself.

### Lesson Learned

For Parquet-mapping reviews, verify that the upstream JSON row can actually differ before treating a repeated source assignment as a live corruption bug. A converter can look wrong in isolation while still matching the already-emitted source struct byte-for-byte.
