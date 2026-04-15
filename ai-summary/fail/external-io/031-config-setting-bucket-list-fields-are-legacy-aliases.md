# H031: Config-setting bucket-list fields should diverge from live Soroban state fields

**Date**: 2026-04-15
**Subsystem**: external-io
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If current Stellar XDR still exposed distinct bucket-list and live-Soroban-state settings, `TransformConfigSetting()` should populate each exported `bucket_list_*` column from its dedicated on-chain source rather than mirroring the `live_soroban_state_*` values. The JSON and Parquet exports should only duplicate these columns when the underlying XDR itself contains identical values.

## Mechanism

I suspected the config-setting transform was still wiring several legacy `bucket_list_*` columns from the live-Soroban-state getters, which would make the export silently report plausible but wrong values for downstream analytics. That would be a meaningful corruption path if the current XDR still carried separate bucket-list fields that the exporter failed to read.

## Trigger

Inspect `TransformConfigSetting()` for mirrored assignments such as `BucketListTargetSizeBytes <- SorobanStateTargetSizeBytes` and `BucketListSizeWindow <- GetLiveSorobanStateSizeWindow()`, then compare them against the pinned XDR definitions for current config-setting entries.

## Target Code

- `internal/transform/config_setting.go:34-62` — mirrored assignments across read-fee and bucket-list-style output fields
- `internal/transform/config_setting.go:85-107,165-172` — sample/window fields all sourced from live-Soroban-state getters
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:59800-59815` — current `ConfigSettingContractLedgerCostV0` only defines disk-read and Soroban-state fields
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:60780-60789` — current `StateArchivalSettings` only defines `LiveSorobanStateSizeWindow*` fields
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:61879-61887` — current union exposes `GetLiveSorobanStateSizeWindow()` and no distinct bucket-list window getter

## Evidence

`TransformConfigSetting()` clearly mirrors several output columns: it assigns `BucketListTargetSizeBytes` from `SorobanStateTargetSizeBytes`, fills both exported windows from `GetLiveSorobanStateSizeWindow()`, and mirrors the sample-size field as well. At first glance that looks like a copy-paste mapping bug.

## Anti-Evidence

The pinned XDR no longer defines distinct bucket-list source fields for these outputs. The generated types expose `SorobanStateTargetSizeBytes`, `LiveSorobanStateSizeWindowSampleSize`, and `GetLiveSorobanStateSizeWindow()`, but there is no current `BucketListTargetSizeBytes`, `BucketListSizeWindowSampleSize`, or `GetBucketListSizeWindow()` source to preserve separately.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated as a JSON/XDR source mismatch

### Why It Failed

The suspicious duplication is coming from legacy output-schema aliases, not from ignored live XDR fields. Current Stellar XDR only exposes the live-Soroban-state settings, so there is no distinct on-chain bucket-list value that stellar-etl could export correctly instead.

### Lesson Learned

Mirrored export columns are only a data-integrity bug when the current upstream schema still carries separate source values. For long-lived BigQuery schemas, legacy alias columns may intentionally mirror the only surviving XDR field, so always verify the generated XDR getters before treating the duplication as corruption.
