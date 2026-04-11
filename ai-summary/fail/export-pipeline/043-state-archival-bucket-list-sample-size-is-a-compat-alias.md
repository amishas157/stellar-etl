# H043: `bucket_list_size_window_sample_size` is wrongly cloned from live Soroban state

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: config-setting field mismatch
**Hypothesis by**: GPT-5.4, high

## Expected Behavior

If `bucket_list_size_window_sample_size` and `live_soroban_state_size_window_sample_size` represent distinct protocol settings, then `export_ledger_entry_changes --export-config-settings` should populate them from distinct XDR fields so downstream config-setting rows can distinguish the two limits.

## Mechanism

`TransformConfigSetting()` assigns both columns from `StateArchivalSettings.LiveSorobanStateSizeWindowSampleSize`, which initially looks like a copy-paste bug that duplicates a live-Soroban value into a legacy bucket-list column. That would make the exported row silently claim two distinct settings are equal.

## Trigger

Export any `CONFIG_SETTING_STATE_ARCHIVAL_SETTINGS` row through `export_ledger_entry_changes --export-config-settings`.

## Target Code

- `internal/transform/config_setting.go:85-93` — `bucketListSizeWindowSampleSize` and `liveSorobanStateSizeWindowSampleSize` both read from the same XDR member
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:60794-60829` — current `StateArchivalSettings` XDR exposes only `LiveSorobanStateSizeWindowSampleSize`

## Evidence

The transform uses two differently named export columns but assigns both from the same source field. That is exactly the shape of several earlier config-setting corruption bugs.

## Anti-Evidence

The current XDR struct has no `BucketListSizeWindowSampleSize` field at all. This is the same Protocol 23 compatibility pattern already seen for other `bucket_list_*` columns that were retained after the protocol renamed them to `live_soroban_state_*`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

There is no distinct live XDR source for `bucket_list_size_window_sample_size`, so the exporter cannot preserve a different on-chain value for that legacy column. The duplication is a compatibility alias, not a fresh export bug.

### Lesson Learned

For config-setting alias candidates, always verify the current generated XDR before assuming two similarly named columns should diverge. If only the renamed `live_soroban_state_*` field still exists on-chain, the legacy `bucket_list_*` column is just a compatibility mirror.
