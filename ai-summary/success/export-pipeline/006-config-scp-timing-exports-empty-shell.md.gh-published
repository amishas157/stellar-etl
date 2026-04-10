# 006: `CONFIG_SETTING_SCP_TIMING` exports as an empty shell row

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformConfigSetting()` silently drops every value from the live XDR arm `CONFIG_SETTING_SCP_TIMING`. When `export_ledger_entry_changes --export-config-settings` encounters that ledger entry, it still emits a plausible row with `config_setting_id=16`, but none of the five SCP timing parameters survive into JSON or Parquet output.

## Root Cause

The transformer reads many older `ConfigSettingEntry` union arms by calling their getters, but it never calls `GetContractScpTiming()`. `ConfigSettingOutput`, `ConfigSettingOutputParquet`, and `ConfigSettingOutput.ToParquet()` also define no destination columns for `ledger_target_close_time_milliseconds`, `nomination_timeout_initial_milliseconds`, `nomination_timeout_increment_milliseconds`, `ballot_timeout_initial_milliseconds`, or `ballot_timeout_increment_milliseconds`, so the data is lost before serialization.

## Reproduction

When a normal ledger-entry-changes export processes a `LedgerEntryTypeConfigSetting` row whose `ConfigSettingId` is `ConfigSettingIdConfigSettingScpTiming`, the command always calls `TransformConfigSetting()` and appends its result to the `config_settings` output. The resulting row preserves only metadata such as `config_setting_id`, `ledger_sequence`, and `closed_at`; the SCP timing values from the source XDR arm are absent from every exported field.

## Affected Code

- `internal/transform/config_setting.go:24-107` — reads many config-setting union arms but never calls `GetContractScpTiming()`
- `internal/transform/config_setting.go:116-178` — constructs the output row from existing fields only, leaving no place for SCP timing values
- `internal/transform/schema.go:563-627` — JSON schema omits all five SCP timing columns
- `internal/transform/schema_parquet.go:305-369` — Parquet schema also omits all five SCP timing columns
- `internal/transform/parquet_converter.go:347-410` — Parquet conversion can only copy the existing incomplete JSON schema
- `cmd/export_ledger_entry_changes.go:245-257` — exports every config-setting entry through `TransformConfigSetting()` without filtering out unsupported arms
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:61051-61057` — upstream XDR defines the five-field `ConfigSettingScpTiming` struct
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:62019-62027` — upstream XDR exposes `GetContractScpTiming()`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestConfigSettingScpTimingExportsEmptyShell`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go build ./... && go test ./internal/transform/... -run TestConfigSettingScpTimingExportsEmptyShell -v`.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestConfigSettingScpTimingExportsEmptyShell(t *testing.T) {
	scpTiming := xdr.ConfigSettingScpTiming{
		LedgerTargetCloseTimeMilliseconds:      111111,
		NominationTimeoutInitialMilliseconds:   222222,
		NominationTimeoutIncrementMilliseconds: 333333,
		BallotTimeoutInitialMilliseconds:       444444,
		BallotTimeoutIncrementMilliseconds:     555555,
	}

	configSetting := xdr.ConfigSettingEntry{
		ConfigSettingId:   xdr.ConfigSettingIdConfigSettingScpTiming,
		ContractScpTiming: &scpTiming,
	}

	gotTiming, ok := configSetting.GetContractScpTiming()
	if !ok {
		t.Fatal("expected ConfigSettingEntry to expose ContractScpTiming via XDR getter")
	}
	if gotTiming != scpTiming {
		t.Fatalf("GetContractScpTiming() = %#v, want %#v", gotTiming, scpTiming)
	}

	ledgerEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 7777777,
		Data: xdr.LedgerEntryData{
			Type:          xdr.LedgerEntryTypeConfigSetting,
			ConfigSetting: &configSetting,
		},
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeConfigSetting,
		Post:       &ledgerEntry,
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{CloseTime: 1000},
			LedgerSeq: 999999,
		},
	}

	output, err := TransformConfigSetting(change, header)
	if err != nil {
		t.Fatalf("TransformConfigSetting() error = %v", err)
	}

	if output.ConfigSettingId != int32(xdr.ConfigSettingIdConfigSettingScpTiming) {
		t.Fatalf("ConfigSettingId = %d, want %d", output.ConfigSettingId, xdr.ConfigSettingIdConfigSettingScpTiming)
	}

	payload, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("json.Marshal() error = %v", err)
	}

	var decoded map[string]any
	if err := json.Unmarshal(payload, &decoded); err != nil {
		t.Fatalf("json.Unmarshal() error = %v", err)
	}

	for _, want := range []float64{111111, 222222, 333333, 444444, 555555} {
		if jsonContainsNumber(decoded, want) {
			t.Fatalf("output unexpectedly retained SCP timing value %v in %+v", want, decoded)
		}
	}
}

func jsonContainsNumber(v any, want float64) bool {
	switch value := v.(type) {
	case map[string]any:
		for _, nested := range value {
			if jsonContainsNumber(nested, want) {
				return true
			}
		}
	case []any:
		for _, nested := range value {
			if jsonContainsNumber(nested, want) {
				return true
			}
		}
	case float64:
		return value == want
	}

	return false
}
```

## Expected vs Actual Behavior

- **Expected**: A `CONFIG_SETTING_SCP_TIMING` export row should preserve the five SCP timing parameters from the source XDR arm.
- **Actual**: The exporter emits a row with `config_setting_id=16`, but none of the five timing values exist in the serialized output because the arm is neither transformed nor represented in the schema.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC constructs the real `ConfigSettingScpTiming` XDR arm, confirms the SDK getter returns the populated values, then runs the production transformer and shows those values never appear in output.
2. Realistic preconditions: YES — `export_ledger_entry_changes` transforms every `LedgerEntryTypeConfigSetting` entry through this path in normal operation.
3. Bug vs by-design: BUG — config-setting exports are intentionally sparse union rows for represented arms, but silently emitting a row for an unrepresented live arm is incomplete export behavior, not an intentional omission signal.
4. Final severity: High — this is structural corruption of exported network-configuration data rather than a financial amount miscalculation.
5. In scope: YES — it is a concrete production code path that produces silently incomplete output.
6. Test correctness: CORRECT — the test uses the real transformer, a distinctive set of sentinel timing values, and a control check proving the source XDR arm is actually populated.
7. Alternative explanations: NONE — there is no other JSON or Parquet field that aliases or stores these five timing values.
8. Novelty: NOVEL

## Suggested Fix

Add five SCP timing fields to `ConfigSettingOutput`, `ConfigSettingOutputParquet`, and `ConfigSettingOutput.ToParquet()`, populate them from `configSetting.GetContractScpTiming()` in `TransformConfigSetting()`, and extend config-setting tests so `CONFIG_SETTING_SCP_TIMING` exports a populated sparse-union row instead of a metadata-only shell.
