# H003: Config-setting export zeroes `ContractScpTiming` rows

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `config_setting_id = 16` (`ConfigSettingIdConfigSettingScpTiming`), the export should preserve the SCP timing parameters — ledger target close time plus the nomination/ballot timeout values in milliseconds. Those network-tuning values should survive into the output row rather than disappearing behind a mostly zero-valued config-settings record.

## Mechanism

The union arm for SCP timing is present in current XDR, but the transform never calls the corresponding getter and the schema defines no SCP timing columns. As a result, an exported config-settings row for ID 16 contains the ID and generic metadata only; all five arm-specific timing fields are silently lost.

## Trigger

Export a config-setting ledger entry with:

1. `ConfigSettingId = ConfigSettingIdConfigSettingScpTiming`
2. One or more non-zero timing fields, such as `LedgerTargetCloseTimeMilliseconds = 5000`

The exporter will emit a row for `config_setting_id = 16` with no way to recover the timing payload from the JSON/Parquet columns.

## Target Code

- `internal/transform/config_setting.go:24-107` — transform reads older config-setting arms only and never extracts SCP timing
- `internal/transform/config_setting.go:116-178` — row construction has no SCP timing fields
- `internal/transform/schema.go:564-627` — `ConfigSettingOutput` contains no columns for the SCP timing payload
- `cmd/export_ledger_entry_changes.go:245-256` — the export path still writes every config-setting entry it receives
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:61011-61017` — `ConfigSettingScpTiming` defines five millisecond timing fields
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:61434-61435` — config-setting ID 16 selects the `ContractScpTiming` arm
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:61556-61562` — the union stores the arm as `ContractScpTiming`, but the transform never reads it

## Evidence

This is not a theoretical future field: the current SDK already generates the concrete `ConfigSettingScpTiming` type and wires it into `ConfigSettingEntry`. The ETL schema stops before those fields, so any SCP timing row exported today is guaranteed to be incomplete.

## Anti-Evidence

If no ledger in the chosen range contains `config_setting_id = 16`, the corruption remains dormant. But the bug is structural: once such a row appears, the transform will silently flatten it because there is no column or extraction path for the arm.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/export-pipeline/006-config-scp-timing-exports-empty-shell.md.gh-published`
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same bug as the already-published finding `export-pipeline/006-config-scp-timing-exports-empty-shell`. Both identify that `TransformConfigSetting()` never calls `GetContractScpTiming()`, that `ConfigSettingOutput` has no fields for the five SCP timing parameters (`ledger_target_close_time_milliseconds`, `nomination_timeout_initial_milliseconds`, `nomination_timeout_increment_milliseconds`, `ballot_timeout_initial_milliseconds`, `ballot_timeout_increment_milliseconds`), and that config_setting_id=16 rows are emitted as empty shells with only metadata. The published finding already includes a complete PoC test, adversarial review, and suggested fix.

### Code Paths Examined

- `internal/transform/config_setting.go:24-107` — confirmed no call to `GetContractScpTiming()`
- `internal/transform/schema.go:563-627` — confirmed no SCP timing fields in `ConfigSettingOutput`
- `ai-summary/success/export-pipeline/006-config-scp-timing-exports-empty-shell.md.gh-published` — identical root cause, mechanism, affected code, and impact

### Why It Failed

This is a duplicate of an already-confirmed and published finding. The published success file `export-pipeline/006-config-scp-timing-exports-empty-shell` covers the identical root cause (missing `GetContractScpTiming()` call), the identical affected schema (no destination columns for the five SCP timing fields), and the identical impact (empty shell rows for config_setting_id=16). No new information is contributed by this hypothesis.

### Lesson Learned

Cross-check `ai-summary/success/` across all subsystems (not just the target subsystem) before submitting, as the same underlying bug may have been discovered and published under a different subsystem classification (here `export-pipeline` vs `data-transform`).
