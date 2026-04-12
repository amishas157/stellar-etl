# H061: Bucket-list ledger-cost columns looked cross-wired, but current XDR exposes only the renamed Soroban-state fields

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: config-setting field mapping
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `bucket_list_target_size_bytes`, `write_fee_1kb_bucket_list_low`, `write_fee_1kb_bucket_list_high`, and `bucket_list_write_fee_growth_factor` represented distinct live network settings, `TransformConfigSetting()` should read separate upstream XDR fields for those columns instead of cloning the Soroban-state rent fields.

## Mechanism

`TransformConfigSetting()` populates both the legacy `bucket_list_*` columns and the newer `soroban_state_*` columns from the same `ConfigSettingContractLedgerCostV0` fields. At first glance that looks like a copy/paste mapping bug because names such as `write_fee_1kb_bucket_list_low` are fed from `RentFee1KbSorobanStateSizeLow`.

## Trigger

Export any `CONFIG_SETTING_CONTRACT_LEDGER_COST_V0` row and inspect the paired legacy/new columns. The bucket-list columns will exactly mirror the Soroban-state columns.

## Target Code

- `internal/transform/config_setting.go:54-61` — duplicates `SorobanState*` / `RentFee1KbSorobanStateSize*` values into both legacy and renamed columns
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:59779-59794` — current XDR struct exposes only `SorobanState*`-named fields

## Evidence

The local transform plainly feeds both column families from the same four upstream values, and the current generated XDR no longer contains any separate `BucketList*` source fields to read from. Without extra context, that looks like a live mis-mapping.

## Anti-Evidence

The codebase already carries several protocol-23 backward-compatibility mirrors where legacy `bucket_list_*` columns intentionally shadow renamed `soroban_state_*` XDR fields. The absence of any distinct upstream `BucketList*` source strongly suggests these columns are compatibility aliases rather than independent settings.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is another Protocol-23 compatibility mirror, not a live field-mapping bug. The generated XDR now exposes only `SorobanStateTargetSizeBytes`, `RentFee1KbSorobanStateSize{Low,High}`, and `SorobanStateRentFeeGrowthFactor`, so there is no separate on-chain bucket-list value available to export.

### Lesson Learned

For config-setting audits, a suspicious same-source mapping is only viable if the current XDR still exposes two distinct fields. When a legacy column family survives a protocol rename without any independent upstream source, the duplication is a compatibility contract, not corruption.
