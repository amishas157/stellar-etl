# H002: `CONTRACT_LEDGER_COST_EXT_V0` drops `tx_max_footprint_entries`

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a config-setting row has `config_setting_id = CONFIG_SETTING_CONTRACT_LEDGER_COST_EXT_V0`, the export should preserve both values from the active XDR arm: `fee_write_1kb` and `tx_max_footprint_entries`. Downstream analytics should be able to recover the on-chain footprint-entry cap from the exported config row.

## Mechanism

`TransformConfigSetting()` does call `GetContractLedgerCostExt()`, but it only reads `FeeWrite1Kb` from that arm and never reads `TxMaxFootprintEntries`. The JSON and Parquet schemas also have no column for that field, so the exporter emits a plausible config-setting row with the correct arm ID while silently discarding the footprint cap entirely.

## Trigger

Run `export_ledger_entry_changes --export-config-settings` on ledgers that contain a `CONFIG_SETTING_CONTRACT_LEDGER_COST_EXT_V0` entry.

## Target Code

- `internal/transform/config_setting.go:34-52` — ext arm is loaded, but only `FeeWrite1Kb` is extracted
- `internal/transform/schema.go:563-627` — `ConfigSettingOutput` has no `tx_max_footprint_entries` field
- `internal/transform/schema_parquet.go:305-369` — Parquet schema likewise has no footprint-entry-cap column

## Evidence

Current XDR exposes `ConfigSettingContractLedgerCostExtV0{TxMaxFootprintEntries, FeeWrite1Kb}`. In the transformer, `contractLedgerCostV0` is populated from `GetContractLedgerCostExt()`, yet only `FeeWrite1Kb` is copied into the output struct. Because the output schemas lack any destination field for `TxMaxFootprintEntries`, the value cannot survive either JSON or Parquet export.

## Anti-Evidence

The row still carries the correct `config_setting_id`, and `fee_write_1kb` is preserved, so consumers can at least detect which arm was active. But the missing footprint cap is the value analytics actually need from that arm, and there is no alternate exported column that recovers it.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced the complete code path from the XDR type definition through the transform function to the output schema. The XDR struct `ConfigSettingContractLedgerCostExtV0` (confirmed in the pinned SDK at `go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go`) has exactly two fields: `TxMaxFootprintEntries Uint32` and `FeeWrite1Kb Int64`. In `config_setting.go:35`, `GetContractLedgerCostExt()` is called and the result is stored in `contractLedgerCostV0`. On line 52, only `contractLedgerCostV0.FeeWrite1Kb` is read. `TxMaxFootprintEntries` is never referenced anywhere in the transform code, and `ConfigSettingOutput` has no corresponding field.

### Code Paths Examined

- `go-stellar-sdk/xdr/xdr_generated.go:ConfigSettingContractLedgerCostExtV0` — XDR struct defines two fields: `TxMaxFootprintEntries Uint32` ("Maximum number of RO+RW entries in the transaction footprint") and `FeeWrite1Kb Int64` ("Fee per 1 KB of data written to the ledger")
- `internal/transform/config_setting.go:35` — `contractLedgerCostV0, _ := configSetting.GetContractLedgerCostExt()` retrieves the extension struct
- `internal/transform/config_setting.go:52` — `feeWrite1Kb := contractLedgerCostV0.FeeWrite1Kb` is the ONLY field read from the extension; `TxMaxFootprintEntries` is never accessed
- `internal/transform/config_setting.go:139` — `FeeWrite1Kb: int64(feeWrite1Kb)` is assigned into the output struct
- `internal/transform/schema.go:563-627` — `ConfigSettingOutput` has no `TxMaxFootprintEntries` or `tx_max_footprint_entries` field
- `internal/transform/schema_parquet.go` — Parquet schema likewise has no column for this value

### Findings

The bug is confirmed. `ConfigSettingContractLedgerCostExtV0` is a two-field XDR struct, and the transform code only extracts one of its two fields. Unlike the backward-compatibility aliasing pattern found in H004 (where `BucketList*` columns mirror renamed `Soroban*` fields), `TxMaxFootprintEntries` has no equivalent or alias in any other config setting type — it is a unique field that only exists on this extension arm. The footprint-entry cap is a distinct network parameter (total RO+RW footprint entries per transaction) that is separate from the individual read/write entry limits exported from `ConfigSettingContractLedgerCostV0` (e.g., `TxMaxDiskReadEntries`, `TxMaxWriteLedgerEntries`). No existing output column captures this value.

### PoC Guidance

- **Test file**: `internal/transform/config_setting_test.go`
- **Setup**: Create a `ConfigSettingEntry` with `ConfigSettingId = CONFIG_SETTING_CONTRACT_LEDGER_COST_EXT_V0` and set `TxMaxFootprintEntries` to a non-zero value (e.g., 100). Call `TransformConfigSetting()`.
- **Steps**: Inspect the returned `ConfigSettingOutput` struct for any field containing the `TxMaxFootprintEntries` value.
- **Assertion**: Assert that no field in `ConfigSettingOutput` contains the value 100 (or whatever was set), confirming the data is lost. A fix would add a `TxMaxFootprintEntries uint32 json:"tx_max_footprint_entries"` field to `ConfigSettingOutput` (and its Parquet equivalent), extract `contractLedgerCostV0.TxMaxFootprintEntries` in the transform, and assign it to the new field.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestConfigSettingLedgerCostExtDropsFootprintCap"
**Test Language**: Go

### Demonstration

The test constructs a `ConfigSettingEntry` with `ConfigSettingIdConfigSettingContractLedgerCostExtV0` (ID=15), setting `TxMaxFootprintEntries=12345` and `FeeWrite1Kb=9999`. After calling `TransformConfigSetting()`, the output's `FeeWrite1Kb` correctly contains 9999 (proving the arm was recognized), but the entire JSON-serialized output contains no trace of the value 12345. This confirms that `TxMaxFootprintEntries` is silently discarded because `ConfigSettingOutput` has no field for it and the transform function never reads it.

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
	// 1. Construct a ConfigSettingEntry with the ledger-cost-ext arm active
	//    and TxMaxFootprintEntries set to a distinctive non-zero value.
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
			ScpValue: xdr.StellarValue{CloseTime: 1000},
			LedgerSeq: 10,
		},
	}

	// 2. Run the production transform.
	output, err := TransformConfigSetting(change, header)
	if err != nil {
		t.Fatalf("TransformConfigSetting returned unexpected error: %v", err)
	}

	// Verify FeeWrite1Kb IS preserved (so the arm was recognized).
	if output.FeeWrite1Kb != 9999 {
		t.Fatalf("FeeWrite1Kb not preserved: got %d, want 9999", output.FeeWrite1Kb)
	}

	// 3. Serialize the entire output to JSON and search for the footprint-cap
	//    value. If TxMaxFootprintEntries (12345) appears nowhere in the JSON,
	//    the data is silently lost — confirming the bug.
	jsonBytes, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("failed to marshal output: %v", err)
	}
	jsonStr := string(jsonBytes)

	// The test PASSES (no error) when the value is absent, proving the bug.
	// If the value were somehow preserved, the test would FAIL, disproving
	// the hypothesis.
	if strings.Contains(jsonStr, "12345") {
		t.Errorf("TxMaxFootprintEntries (12345) unexpectedly found in output — hypothesis disproved.\n"+
			"JSON output: %s", jsonStr)
	}

	t.Logf("BUG CONFIRMED: TxMaxFootprintEntries=12345 set on XDR input is absent from output.\n"+
		"FeeWrite1Kb=9999 from the same arm IS preserved (config_setting_id=15).\n"+
		"JSON output: %s", jsonStr)
}
```

### Test Output

```
=== RUN   TestConfigSettingLedgerCostExtDropsFootprintCap
    data_integrity_poc_test.go:79: BUG CONFIRMED: TxMaxFootprintEntries=12345 set on XDR input is absent from output.
        FeeWrite1Kb=9999 from the same arm IS preserved (config_setting_id=15).
        JSON output: {"config_setting_id":15,"contract_max_size_bytes":0,...,"fee_write_1kb":9999,...,"last_modified_ledger":100,"ledger_entry_change":0,"deleted":false,"closed_at":"1970-01-01T00:16:40Z","ledger_sequence":10}
--- PASS: TestConfigSettingLedgerCostExtDropsFootprintCap (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.969s
```
