# H003: `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` exports as a metadata-only shell row

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-config-settings` encounters `ConfigSettingIdConfigSettingContractParallelComputeV0`, the emitted row should preserve the arm's on-chain value `ledger_max_dependent_tx_clusters` in JSON and Parquet output. The row should contain both the metadata fields (`config_setting_id`, `ledger_sequence`, `closed_at`, etc.) and the actual parallel-compute parameter.

## Mechanism

The generated XDR now exposes a dedicated `ContractParallelCompute` union arm with `LedgerMaxDependentTxClusters`, but `TransformConfigSetting()` never calls `GetContractParallelCompute()` and the export schemas define no destination column for that value. The exporter still emits a config-setting row for the ledger entry, so this arm becomes a plausible metadata-only shell that silently drops the only business field it carries.

## Trigger

Run `export_ledger_entry_changes --export-config-settings` over a ledger containing `ConfigSettingIdConfigSettingContractParallelComputeV0` (config-setting ID 14). The exported row will keep metadata like `config_setting_id=14` while omitting the arm's `ledger_max_dependent_tx_clusters` value entirely.

## Target Code

- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/xdr_generated.go:59683-59690` — `ConfigSettingContractParallelComputeV0` defines `LedgerMaxDependentTxClusters`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/xdr_generated.go:61957-61979` — XDR getter exposes the `ContractParallelCompute` arm
- `internal/transform/config_setting.go:26-179` — transformer reads older union arms but never calls `GetContractParallelCompute()`
- `internal/transform/schema.go:564-627` — config-setting schema has no `ledger_max_dependent_tx_clusters` column

## Evidence

This is the same omission pattern already confirmed for `tx_max_footprint_entries` and `CONFIG_SETTING_SCP_TIMING`: a live XDR getter exists, the export command transforms every config-setting ledger entry through the same wide-row path, and the destination schemas simply have no place to put the arm's value. The current transformer stops at `GetContractExecutionLanes()` and never touches the newer parallel-compute getter.

## Anti-Evidence

If current public networks do not emit this union arm yet, the corruption remains latent until such a ledger appears. But once a ledger does contain config-setting ID 14, the loss is deterministic because there is no matching field anywhere in the export pipeline.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `TransformConfigSetting()` in `config_setting.go` through all union arm getter calls. The function calls `GetContractMaxSizeBytes()`, `GetContractCompute()`, `GetContractLedgerCost()`, `GetContractLedgerCostExt()`, `GetContractHistoricalData()`, `GetContractEvents()`, `GetContractBandwidth()`, `GetContractCostParamsCpuInsns()`, `GetContractCostParamsMemBytes()`, `GetContractDataKeySizeBytes()`, `GetContractDataEntrySizeBytes()`, `GetStateArchivalSettings()`, `GetContractExecutionLanes()`, and `GetLiveSorobanStateSizeWindow()` — but never `GetContractParallelCompute()`. The `ConfigSettingOutput` struct (schema.go:564-627) has no field for `ledger_max_dependent_tx_clusters`. Confirmed the XDR type exists in the actual project SDK (`v0.0.0-20251211085638-ba09a6a91775`) with `GetContractParallelCompute()` getter at line 61971 and `LedgerMaxDependentTxClusters Uint32` at line 59684.

### Code Paths Examined

- `internal/transform/config_setting.go:26-107` — reads 14 union arm getters but never calls `GetContractParallelCompute()`; last arm accessed is `GetContractExecutionLanes()` at line 98
- `internal/transform/config_setting.go:116-178` — constructs `ConfigSettingOutput` from extracted values; no parallel-compute field populated
- `internal/transform/schema.go:564-627` — `ConfigSettingOutput` struct defines ~60 fields but no `ledger_max_dependent_tx_clusters` or equivalent
- `go-stellar-sdk/xdr/xdr_generated.go:59684` — `LedgerMaxDependentTxClusters Uint32` field exists on `ConfigSettingContractParallelComputeV0`
- `go-stellar-sdk/xdr/xdr_generated.go:61969-61976` — `GetContractParallelCompute()` getter returns the populated arm

### Findings

This is the exact same bug class as the already-confirmed success finding `006-config-scp-timing-exports-empty-shell`. When a ledger contains `ConfigSettingIdConfigSettingContractParallelComputeV0` (ID 14), the transformer emits a row with metadata (`config_setting_id=14`, `ledger_sequence`, `closed_at`) but silently drops the only business field (`ledger_max_dependent_tx_clusters`) because:

1. `TransformConfigSetting()` never calls `GetContractParallelCompute()`
2. `ConfigSettingOutput` has no destination field for the value
3. `ConfigSettingOutputParquet` likewise has no destination field
4. The export path in `export_ledger_entry_changes` transforms every config-setting entry without filtering unsupported arms

The existing test file (`config_setting_test.go`) has no test case exercising this union arm.

### PoC Guidance

- **Test file**: `internal/transform/config_setting_test.go` (or `internal/transform/data_integrity_poc_test.go` if it exists)
- **Setup**: Construct a `ConfigSettingEntry` with `ConfigSettingId: xdr.ConfigSettingIdConfigSettingContractParallelComputeV0` and `ContractParallelCompute` set to a `ConfigSettingContractParallelComputeV0{LedgerMaxDependentTxClusters: 42}`. Wrap in a `LedgerEntry` and `ingest.Change`.
- **Steps**: Call `TransformConfigSetting(change, header)` and JSON-marshal the result.
- **Assertion**: Assert that the JSON output does NOT contain the value `42` for `ledger_max_dependent_tx_clusters` — proving the field is silently dropped. This demonstrates the bug. (Follow the same PoC pattern as success/006.)

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestConfigSettingParallelComputeDropped"
**Test Language**: Go

### Demonstration

The test constructs a `ConfigSettingEntry` with `ConfigSettingIdConfigSettingContractParallelComputeV0` (ID 14) and `LedgerMaxDependentTxClusters=42`, then runs it through `TransformConfigSetting()`. The output row carries `config_setting_id=14` (metadata shell) but the JSON contains no `ledger_max_dependent_tx_clusters` field at all — the value 42 is silently dropped. This confirms the transformer emits a metadata-only shell row for this union arm.

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

// TestConfigSettingParallelComputeDropped demonstrates that TransformConfigSetting
// silently drops the ledger_max_dependent_tx_clusters value for the
// ConfigSettingContractParallelComputeV0 union arm (config setting ID 14).
// The transformer emits a metadata-only shell row with config_setting_id=14 but
// no field carrying the actual on-chain value.
func TestConfigSettingParallelComputeDropped(t *testing.T) {
	// 1. Construct a ConfigSettingEntry for the parallel compute arm with a
	//    distinctive value that we can search for in the JSON output.
	parallelCompute := xdr.ConfigSettingContractParallelComputeV0{
		LedgerMaxDependentTxClusters: xdr.Uint32(42),
	}

	ledgerEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 100,
		Data: xdr.LedgerEntryData{
			Type: xdr.LedgerEntryTypeConfigSetting,
			ConfigSetting: &xdr.ConfigSettingEntry{
				ConfigSettingId:         xdr.ConfigSettingIdConfigSettingContractParallelComputeV0,
				ContractParallelCompute: &parallelCompute,
			},
		},
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeConfigSetting,
		Pre:        nil,
		Post:       &ledgerEntry,
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	// 2. Run the production transform code path
	output, err := TransformConfigSetting(change, header)
	if err != nil {
		t.Fatalf("TransformConfigSetting returned unexpected error: %v", err)
	}

	// Verify the metadata shell is present: config_setting_id should be 14
	if output.ConfigSettingId != 14 {
		t.Fatalf("Expected config_setting_id=14, got %d", output.ConfigSettingId)
	}

	// 3. Marshal to JSON and check if the value 42 is anywhere in the output.
	//    Since there is no field for ledger_max_dependent_tx_clusters, the value
	//    should be completely absent from the serialized output.
	jsonBytes, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("Failed to marshal output: %v", err)
	}

	jsonStr := string(jsonBytes)

	// The value "42" for ledger_max_dependent_tx_clusters should NOT appear
	// as a dedicated field in the JSON output — proving the data is dropped.
	if strings.Contains(jsonStr, `"ledger_max_dependent_tx_clusters"`) {
		t.Fatal("Field ledger_max_dependent_tx_clusters unexpectedly found in output — bug may be fixed")
	}

	// Additionally, confirm that the output struct has no way to carry this value:
	// all numeric fields should be zero (since only metadata + the parallel compute
	// arm was set, and the parallel compute value is dropped).
	// We check a few representative fields that should be zero:
	if output.LedgerMaxTxCount != 0 {
		t.Errorf("LedgerMaxTxCount should be 0 for parallel compute arm, got %d", output.LedgerMaxTxCount)
	}

	t.Logf("BUG CONFIRMED: ConfigSettingContractParallelComputeV0 with LedgerMaxDependentTxClusters=42 "+
		"produced a row with config_setting_id=14 but the value 42 is silently dropped.")
	t.Logf("JSON output: %s", jsonStr)
}
```

### Test Output

```
=== RUN   TestConfigSettingParallelComputeDropped
    data_integrity_poc_test.go:86: BUG CONFIRMED: ConfigSettingContractParallelComputeV0 with LedgerMaxDependentTxClusters=42 produced a row with config_setting_id=14 but the value 42 is silently dropped.
    data_integrity_poc_test.go:88: JSON output: {"config_setting_id":14,"contract_max_size_bytes":0,"ledger_max_instructions":0,"tx_max_instructions":0,"fee_rate_per_instructions_increment":0,"tx_memory_limit":0,"ledger_max_read_ledger_entries":0,"ledger_max_disk_read_entries":0,"ledger_max_read_bytes":0,"ledger_max_disk_read_bytes":0,"ledger_max_write_ledger_entries":0,"ledger_max_write_bytes":0,"tx_max_read_ledger_entries":0,"tx_max_disk_read_entries":0,"tx_max_read_bytes":0,"tx_max_disk_read_bytes":0,"tx_max_write_ledger_entries":0,"tx_max_write_bytes":0,"fee_read_ledger_entry":0,"fee_disk_read_ledger_entry":0,"fee_write_ledger_entry":0,"fee_read_1kb":0,"fee_write_1kb":0,"fee_disk_read_1kb":0,"bucket_list_target_size_bytes":0,"soroban_state_target_size_bytes":0,"write_fee_1kb_bucket_list_low":0,"rent_fee_1kb_soroban_state_size_low":0,"write_fee_1kb_bucket_list_high":0,"rent_fee_1kb_soroban_state_size_high":0,"bucket_list_write_fee_growth_factor":0,"soroban_state_rent_fee_growth_factor":0,"fee_historical_1kb":0,"tx_max_contract_events_size_bytes":0,"fee_contract_events_1kb":0,"ledger_max_txs_size_bytes":0,"tx_max_size_bytes":0,"fee_tx_size_1kb":0,"contract_cost_params_cpu_insns":[],"contract_cost_params_mem_bytes":[],"contract_data_key_size_bytes":0,"contract_data_entry_size_bytes":0,"max_entry_ttl":0,"min_temporary_ttl":0,"min_persistent_ttl":0,"auto_bump_ledgers":0,"persistent_rent_rate_denominator":0,"temp_rent_rate_denominator":0,"max_entries_to_archive":0,"bucket_list_size_window_sample_size":0,"live_soroban_state_size_window_sample_size":0,"live_soroban_state_size_window_sample_period":0,"eviction_scan_size":0,"starting_eviction_scan_level":0,"ledger_max_tx_count":0,"bucket_list_size_window":[],"live_soroban_state_size_window":[],"last_modified_ledger":100,"ledger_entry_change":0,"deleted":false,"closed_at":"1970-01-01T00:16:40Z","ledger_sequence":10}
--- PASS: TestConfigSettingParallelComputeDropped (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.702s
```
