# 005: Config-setting export drops `tx_max_footprint_entries`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformConfigSetting()` recognizes the `CONFIG_SETTING_CONTRACT_LEDGER_COST_EXT_V0` arm and exports `fee_write_1kb`, but it silently drops the arm's other on-chain value: `tx_max_footprint_entries`. Because neither the JSON schema, Parquet schema, nor Parquet converter has a destination for that field, every export of this config-setting arm produces an incomplete but plausible row.

## Root Cause

The transform code reads `contractLedgerCostV0 := configSetting.GetContractLedgerCostExt()` and copies only `FeeWrite1Kb` out of that XDR struct. `ConfigSettingOutput`, `ConfigSettingOutputParquet`, and `ConfigSettingOutput.ToParquet()` do not define or populate any `tx_max_footprint_entries` column, so the footprint cap is lost before either JSON or Parquet serialization.

## Reproduction

When `export_ledger_entry_changes --export-config-settings` processes a ledger containing `CONFIG_SETTING_CONTRACT_LEDGER_COST_EXT_V0`, the exporter emits a row with `config_setting_id=15` and the correct `fee_write_1kb`, but there is no exported field carrying the arm's `tx_max_footprint_entries` value. Downstream consumers cannot recover that network parameter from any other exported column.

## Affected Code

- `internal/transform/config_setting.go:34-52` — loads `ContractLedgerCostExt` but only reads `FeeWrite1Kb`
- `internal/transform/schema.go:563-627` — JSON config-setting schema has no `tx_max_footprint_entries` field
- `internal/transform/schema_parquet.go:305-369` — Parquet schema has no `tx_max_footprint_entries` column
- `internal/transform/parquet_converter.go:347-410` — Parquet conversion copies all existing config-setting fields, but there is no source or destination for the footprint cap

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestConfigSettingLedgerCostExtDropsFootprintCap`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go test ./internal/transform/... -run TestConfigSettingLedgerCostExtDropsFootprintCap -v`.

### Test Body

```go
package transform

import (
	"encoding/json"
	"strings"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestConfigSettingLedgerCostExtDropsFootprintCap demonstrates that
// TransformConfigSetting silently discards TxMaxFootprintEntries from the
// CONFIG_SETTING_CONTRACT_LEDGER_COST_EXT_V0 arm because neither the
// ConfigSettingOutput struct nor the transform function references that field.
func TestConfigSettingLedgerCostExtDropsFootprintCap(t *testing.T) {
	const footprintCap xdr.Uint32 = 12345

	extV0 := xdr.ConfigSettingContractLedgerCostExtV0{
		TxMaxFootprintEntries: footprintCap,
		FeeWrite1Kb:           xdr.Int64(9999),
	}

	entry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 100,
		Data: xdr.LedgerEntryData{
			Type: xdr.LedgerEntryTypeConfigSetting,
			ConfigSetting: &xdr.ConfigSettingEntry{
				ConfigSettingId:       xdr.ConfigSettingIdConfigSettingContractLedgerCostExtV0,
				ContractLedgerCostExt: &extV0,
			},
		},
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeConfigSetting,
		Pre:        nil,
		Post:       &entry,
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue:  xdr.StellarValue{CloseTime: 1000},
			LedgerSeq: 10,
		},
	}

	output, err := TransformConfigSetting(change, header)
	if err != nil {
		t.Fatalf("TransformConfigSetting returned unexpected error: %v", err)
	}

	if output.FeeWrite1Kb != 9999 {
		t.Fatalf("FeeWrite1Kb not preserved: got %d, want 9999", output.FeeWrite1Kb)
	}

	jsonBytes, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("failed to marshal output: %v", err)
	}
	jsonStr := string(jsonBytes)

	if strings.Contains(jsonStr, "12345") {
		t.Errorf("TxMaxFootprintEntries (12345) unexpectedly found in output — hypothesis disproved.\n"+
			"JSON output: %s", jsonStr)
	}

	t.Logf("BUG CONFIRMED: TxMaxFootprintEntries=12345 set on XDR input is absent from output.\n"+
		"FeeWrite1Kb=9999 from the same arm IS preserved (config_setting_id=15).\n"+
		"JSON output: %s", jsonStr)
}
```

## Expected vs Actual Behavior

- **Expected**: A `CONFIG_SETTING_CONTRACT_LEDGER_COST_EXT_V0` export row should preserve both XDR members: `fee_write_1kb` and `tx_max_footprint_entries`.
- **Actual**: The exporter preserves `fee_write_1kb` but drops `tx_max_footprint_entries` entirely from both JSON and Parquet outputs.

## Adversarial Review

1. Exercises claimed bug: YES — the test constructs the real XDR arm, runs `TransformConfigSetting()`, and shows the footprint cap disappears while another field from the same arm survives.
2. Realistic preconditions: YES — any normal config-setting export that encounters `CONFIG_SETTING_CONTRACT_LEDGER_COST_EXT_V0` takes this path.
3. Bug vs by-design: BUG — the upstream XDR arm defines two independent fields, and the exporter already treats this table as a sparse union of config-setting-specific columns rather than intentionally omitting whole arm members.
4. Final severity: High — this is structural data corruption in exported network-configuration data, not a financial amount bug.
5. In scope: YES — it is a concrete production code path that emits silently incomplete export data.
6. Test correctness: CORRECT — it uses the production transform, a distinctive sentinel value, and a control assertion showing the arm itself was recognized via `FeeWrite1Kb`.
7. Alternative explanations: NONE — there is no other JSON or Parquet field that stores or aliases `tx_max_footprint_entries`.
8. Novelty: NOVEL — the code path and reproduced loss are distinct; duplicate handling remains the orchestrator's responsibility.

## Suggested Fix

Add `TxMaxFootprintEntries uint32 json:"tx_max_footprint_entries"` to `ConfigSettingOutput`, add the matching Parquet field, populate it from `contractLedgerCostV0.TxMaxFootprintEntries` in `TransformConfigSetting()`, and extend `ToParquet()` plus tests so both JSON and Parquet exports preserve the value.
