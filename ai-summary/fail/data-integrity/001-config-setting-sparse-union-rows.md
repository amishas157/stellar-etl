# H001: Config-setting rows wrongly zero unrelated columns because union getters fail

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each exported config-setting row should populate only the fields represented by the active `ConfigSettingEntry` union arm and leave unrelated columns at their zero/default values. A `CONFIG_SETTING_CONTRACT_MAX_SIZE_BYTES` row should therefore carry `contract_max_size_bytes` plus metadata, while unrelated compute, bandwidth, and archival fields remain empty.

## Mechanism

I initially suspected the repeated `Get*()` calls in `TransformConfigSetting()` were an integrity bug because most union getters return zero values on any single row. That would only be a bug if the ETL were meant to emit a fully denormalized snapshot per row, rather than a sparse one-row-per-setting export.

## Trigger

Export a `CONFIG_SETTING_CONTRACT_MAX_SIZE_BYTES` entry and observe that non-active fields remain zero.

## Target Code

- `internal/transform/config_setting.go:24-99` — reads every union arm from a single `ConfigSettingEntry`
- `internal/transform/config_setting_test.go:62-156` — test fixture expects a sparse row with zeroed unrelated fields

## Evidence

`ConfigSettingEntry` is an XDR union, so only one arm is populated on any ledger entry. The transformer still probes all getters, which first looked like a sign that most columns were being dropped accidentally.

## Anti-Evidence

The existing test suite explicitly asserts that a `CONTRACT_MAX_SIZE_BYTES` row should export zeros for every unrelated field. That makes the sparse-row behavior intentional in the current design, even if the wide schema is awkward.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The current transformer and its tests model `config_settings` as sparse rows keyed by `config_setting_id`, not as a full snapshot table. Zero values in unrelated columns are therefore expected behavior for the active union-arm design.

### Lesson Learned

Treat `ConfigSettingEntry` exports as one-row-per-setting-arm when evaluating integrity issues. Focus on mislabeled populated columns, not on other columns being left empty.
