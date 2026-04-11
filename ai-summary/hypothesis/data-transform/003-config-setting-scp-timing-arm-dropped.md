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
