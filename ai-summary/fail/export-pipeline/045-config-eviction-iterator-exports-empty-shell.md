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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `data-integrity/006-eviction-iterator-empty-shell.md.gh-published`
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same bug already confirmed and published as `ai-summary/success/data-integrity/006-eviction-iterator-empty-shell.md.gh-published`. The same finding was also previously rejected as duplicate `024a.md` in `ai-summary/fail/data-transform/summary.md` and as entry `041.md` in `ai-summary/fail/export-pipeline/summary.md`. The root cause, affected code paths, missing fields, and impact are identical across all filings.

### Code Paths Examined

- `internal/transform/config_setting.go:85-96` — confirmed `GetStateArchivalSettings()` is called but `GetEvictionIterator()` is not
- `ai-summary/success/data-integrity/006-eviction-iterator-empty-shell.md.gh-published` — confirmed published finding covers this exact issue
- `ai-summary/fail/export-pipeline/summary.md:041` — confirmed this exact hypothesis was previously rejected as a cross-subsystem duplicate
- `ai-summary/fail/data-transform/summary.md:024a` — confirmed another prior duplicate rejection

### Why It Failed

This is a cross-subsystem duplicate of an already-confirmed and published finding (`data-integrity/006-eviction-iterator-empty-shell`). The same missing `GetEvictionIterator()` call, same missing schema columns (`bucket_list_level`, `is_curr_bucket`, `bucket_file_offset`), and same impact (metadata-only shell row for config_setting_id=13) have been thoroughly documented and published. This is the third time this exact finding has been filed under a different subsystem label.

### Lesson Learned

Always check `ai-summary/success/` across ALL subsystems before filing a hypothesis. The eviction-iterator empty-shell bug was originally confirmed under `data-integrity`, not `export-pipeline` or `data-transform`, demonstrating that cross-subsystem duplicate checking is essential.
