# H001: Config-setting export zeroes `EvictionIterator` rows

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-config-settings` encounters a `ConfigSettingEntry` with `config_setting_id = 13` (`ConfigSettingIdConfigSettingEvictionIterator`), the exported row should preserve the arm-specific iterator state: `bucket_list_level`, `is_curr_bucket`, and `bucket_file_offset`. A non-zero `EvictionIterator` entry should not collapse into a row that contains only the ID plus zero-valued legacy columns.

## Mechanism

`TransformConfigSetting()` never calls `GetEvictionIterator()` and `ConfigSettingOutput` has no fields for the arm's three members. For `config_setting_id = 13`, every getter that the transform does call misses the active union arm and returns zero values, so the exporter still emits a row but silently drops the only payload data that entry actually carries.

## Trigger

Process any ledger-entry change whose `ConfigSettingEntry.ConfigSettingId` is `ConfigSettingIdConfigSettingEvictionIterator` and whose XDR payload contains non-default iterator values, for example:

1. `BucketListLevel = 4`
2. `IsCurrBucket = true`
3. `BucketFileOffset = 8192`

The resulting JSON/Parquet row will keep `config_setting_id = 13` but export zeros/empty values for the actual iterator state.

## Target Code

- `internal/transform/config_setting.go:24-107` — transform reads older config-setting arms but never calls `GetEvictionIterator()`
- `internal/transform/config_setting.go:116-178` — emitted row has no destination for iterator fields
- `internal/transform/schema.go:564-627` — `ConfigSettingOutput` defines no `EvictionIterator` columns
- `cmd/export_ledger_entry_changes.go:245-256` — every config-setting ledger entry is routed through `TransformConfigSetting()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:60925-60929` — `EvictionIterator` contains `BucketListLevel`, `IsCurrBucket`, and `BucketFileOffset`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:61428-61429` — config-setting ID 13 selects the `EvictionIterator` union arm
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:61906-61914` — SDK exposes `GetEvictionIterator()`, but the transform never uses it

## Evidence

The current transform is written as a flat field grabber for the older config-setting arms only. Once the active union arm is `EvictionIterator`, the function still constructs a `ConfigSettingOutput`, but none of the row fields correspond to the arm's payload. The checked-in test only covers `ConfigSettingId = 0`, so this path is presently untested.

## Anti-Evidence

If the exported ledger range never includes `config_setting_id = 13`, the corruption stays latent. But the current XDR enum already defines the arm and the export command does not filter it out, so any network that emits this config setting will produce plausibly shaped but payload-free rows.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-integrity/006-eviction-iterator-empty-shell.md.gh-published`
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same bug as the already-published finding `data-integrity/006-eviction-iterator-empty-shell`. Both identify that `TransformConfigSetting()` never calls `GetEvictionIterator()`, that `ConfigSettingOutput` has no fields for `bucket_list_level`, `is_curr_bucket`, or `bucket_file_offset`, and that config_setting_id=13 rows are emitted as empty shells with only metadata. The published finding already includes a complete PoC test and suggested fix.

### Code Paths Examined

- `internal/transform/config_setting.go:24-107` — confirmed no call to `GetEvictionIterator()`
- `internal/transform/schema.go:563-627` — confirmed no eviction-iterator fields in `ConfigSettingOutput`
- `ai-summary/success/data-integrity/006-eviction-iterator-empty-shell.md.gh-published` — identical root cause, mechanism, affected code, and impact

### Why It Failed

This is a duplicate of an already-confirmed and published finding. The published success file `data-integrity/006-eviction-iterator-empty-shell` covers the identical root cause (missing `GetEvictionIterator()` call), the identical affected schema (no destination columns for iterator fields), and the identical impact (empty shell rows for config_setting_id=13). No new information is contributed by this hypothesis.

### Lesson Learned

Cross-check `ai-summary/success/` across all subsystems (not just the target subsystem) before submitting, as the same underlying bug may have been discovered and published under a different subsystem classification (here `data-integrity` vs `data-transform`).
