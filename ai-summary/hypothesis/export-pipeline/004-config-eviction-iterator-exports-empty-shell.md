# H004: `CONFIG_SETTING_EVICTION_ITERATOR` exports as a metadata-only shell row

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When the config-setting union arm is `ConfigSettingIdConfigSettingEvictionIterator`, the export should preserve the iterator state fields `bucket_list_level`, `is_curr_bucket`, and `bucket_file_offset`. Downstream consumers should be able to recover the iterator position from the emitted row rather than only seeing row metadata.

## Mechanism

The generated XDR exposes a dedicated `EvictionIterator` struct and getter, but `TransformConfigSetting()` never calls `GetEvictionIterator()` and the schemas define no columns for any of the iterator fields. The transformer instead exports only the older `StateArchivalSettings` summary fields (`eviction_scan_size`, `starting_eviction_scan_level`), so a real `EvictionIterator` ledger entry is reduced to a plausible shell row with its actual payload stripped out.

## Trigger

Run `export_ledger_entry_changes --export-config-settings` over a ledger containing `ConfigSettingIdConfigSettingEvictionIterator` (config-setting ID 13). The exporter will emit a row with metadata such as `config_setting_id=13` but no `bucket_list_level`, `is_curr_bucket`, or `bucket_file_offset` data.

## Target Code

- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/xdr_generated.go:60965-60969` — `EvictionIterator` carries `BucketListLevel`, `IsCurrBucket`, and `BucketFileOffset`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/xdr_generated.go:61944-61955` — XDR getter exposes the `EvictionIterator` arm
- `internal/transform/config_setting.go:85-99` — transformer reads `StateArchivalSettings` eviction summary fields but never reads the dedicated iterator arm
- `internal/transform/schema.go:564-627` — config-setting export schema has no iterator-position columns

## Evidence

The exporter already emits every config-setting entry through the same sparse wide-row path, and previous confirmed findings showed that missing getters on this path surface as metadata-only shell rows rather than hard failures. Here the omission is even clearer: the XDR arm has three fields, none appear in the schema, and the transformer never references `GetEvictionIterator()`.

## Anti-Evidence

This issue only manifests when ledgers actually contain the dedicated `EvictionIterator` arm; older networks or test fixtures may never hit it. The presence of related archival fields in `StateArchivalSettings` could also mislead reviewers into thinking the iterator is already covered, even though those fields describe different data.
