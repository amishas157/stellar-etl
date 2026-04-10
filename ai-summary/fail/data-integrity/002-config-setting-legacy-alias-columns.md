# H002: Config-setting "read" and "bucket_list" columns are accidentally copied from the wrong XDR fields

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the config-setting schema exposed distinct on-chain values for `ledger_max_read_*`, `fee_read_*`, `bucket_list_*`, and `live_soroban_state_*`, each exported column should be populated from its own XDR member. A `CONFIG_SETTING_CONTRACT_LEDGER_COST_V0` or `CONFIG_SETTING_STATE_ARCHIVAL` row would then show different values whenever the protocol carried genuinely different read-vs-disk or bucket-list-vs-live-state settings.

## Mechanism

I initially suspected a copy-paste corruption bug because `TransformConfigSetting()` assigns several `*_read_*` and `bucket_list_*` outputs from `DiskRead*` and `LiveSorobanState*` sources. That would be a real bug only if the XDR union actually contained separate source members for the aliased columns.

## Trigger

Export config-setting rows for `CONFIG_SETTING_CONTRACT_LEDGER_COST_V0`, `CONFIG_SETTING_STATE_ARCHIVAL`, or `CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW` and compare the duplicated output columns.

## Target Code

- `internal/transform/config_setting.go:34-62` — fills `ledger_max_read_*`, `fee_read_*`, and `bucket_list_*` outputs from disk/live-state members
- `internal/transform/config_setting.go:85-107` — fills both sample/window aliases from `StateArchivalSettings` and `GetLiveSorobanStateSizeWindow()`
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:59779-59795` — `ConfigSettingContractLedgerCostV0` only defines `DiskRead*`, `FeeDiskRead*`, and `SorobanState*` members
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:60818-60829` — `StateArchivalSettings` only defines `LiveSorobanStateSizeWindowSampleSize`, not a bucket-list counterpart
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:61400-61429` — `ConfigSettingEntry` has a `LiveSorobanStateSizeWindow` arm but no separate `BucketListSizeWindow` arm

## Evidence

The transform layer duplicates values into pairs of similarly named output columns, which strongly resembles a field-mapping bug at first glance. The schema itself also contains both legacy-style `bucket_list_*` names and newer Soroban-state names.

## Anti-Evidence

The generated XDR types do not expose separate source fields for the suspected aliases. There is no on-chain `LedgerMaxReadLedgerEntries`, `FeeReadLedgerEntry`, `BucketListSizeWindow`, or `BucketListSizeWindowSampleSize` member to export independently from the `DiskRead*` and `LiveSorobanState*` values.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The apparent duplicates are legacy schema aliases over a single on-chain source, not a transform bug. The current XDR union only carries `DiskRead*` and `LiveSorobanState*` values, so the ETL cannot populate distinct values for the aliased columns without inventing data.

### Lesson Learned

For `config_settings`, verify the generated XDR union before treating mirrored columns as copy-paste corruption. Several wide-schema columns are compatibility aliases whose duplication is driven by the export contract, not by missing source reads.
