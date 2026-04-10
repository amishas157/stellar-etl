# H009: Bucket-list-prefixed config-setting aliases do not currently have distinct XDR backing fields

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If config-setting output columns such as `bucket_list_target_size_bytes`, `write_fee_1kb_bucket_list_low`, or `bucket_list_size_window` represent distinct protocol values, `export_ledger_entry_changes` should read them from distinct XDR fields rather than duplicating Soroban-state values.

## Mechanism

`TransformConfigSetting` duplicates several Soroban-state values into bucket-list-prefixed columns and also fills both `BucketListSizeWindow` and `LiveSorobanStateSizeWindow` from `GetLiveSorobanStateSizeWindow()`. That initially looked like a broad copy-paste mapping bug in the config-setting export path.

## Trigger

Run `export_ledger_entry_changes --export-config-settings` on ledgers containing Soroban config-setting entries and inspect the bucket-list-prefixed columns for rows with non-zero values.

## Target Code

- `internal/transform/config_setting.go:34-61` — bucket-list-prefixed scalar columns are populated from Soroban-state fields
- `internal/transform/config_setting.go:101-107` — both exported windows are populated from `GetLiveSorobanStateSizeWindow()`
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:59742-59795` — `ConfigSettingContractLedgerCostV0` exposes only Soroban-state target size / rent-fee fields, not separate bucket-list fields
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:61411-61930` — `ConfigSettingEntry` exposes `GetLiveSorobanStateSizeWindow()` but no separate bucket-list window arm

## Evidence

The local transform code duplicates multiple values into differently named output columns, which is exactly the shape of a real copy-paste bug. The output schema also presents both bucket-list and live-state columns, so the mismatch is easy to suspect when reading `config_setting.go`.

## Anti-Evidence

The current XDR definitions do not provide distinct bucket-list-backed source fields for these columns. `ConfigSettingContractLedgerCostV0` only exposes Soroban-state target size and rent-fee fields, and `ConfigSettingEntry` only exposes a `LiveSorobanStateSizeWindow` arm. That means the duplicated values appear to be legacy compatibility aliases rather than a case where stellar-etl is ignoring available distinct on-chain data.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspicious duplication does not correspond to distinct source fields in the current Stellar XDR. Because the protocol no longer provides separate bucket-list values for these settings, the transform is not currently overwriting one real on-chain value with another.

### Lesson Learned

Config-setting code contains legacy alias columns whose names no longer match distinct protocol fields. Before calling those aliases corruption, confirm that the current XDR union still exposes separate source arms or struct members for the allegedly different values.
