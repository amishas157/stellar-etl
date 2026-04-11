# H001: `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` exports an empty shell

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-config-settings` encounters a
`CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` ledger entry, the exported
`config_settings` row should preserve the on-chain
`ledger_max_dependent_tx_clusters` value in both JSON and Parquet output.
Downstream consumers should be able to distinguish a real configured cluster
limit from an unset or zero value.

## Mechanism

Upstream XDR already exposes the `ContractParallelCompute` union arm and its
`LedgerMaxDependentTxClusters` field, but `TransformConfigSetting()` never
calls `GetContractParallelCompute()`. `ConfigSettingOutput` and
`ConfigSettingOutputParquet` also define no destination column for that value,
so the exporter still emits a plausible sparse-union row with
`config_setting_id=14` and normal metadata while silently dropping the only
meaningful payload from the ledger entry.

## Trigger

1. Run `export_ledger_entry_changes --export-config-settings` over any ledger
   range that contains `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0`.
2. Inspect the emitted `config_settings` row with `config_setting_id=14`.
3. Observe that the row contains only generic metadata/sparse-union defaults,
   even though the source XDR arm contains a non-zero
   `ledger_max_dependent_tx_clusters` value.

## Target Code

- `internal/transform/config_setting.go:13-179` — reads many config-setting
  union arms, but never `GetContractParallelCompute()`
- `internal/transform/schema.go:563-627` — `ConfigSettingOutput` has no column
  for `ledger_max_dependent_tx_clusters`
- `internal/transform/schema_parquet.go:305-369` — Parquet schema also lacks a
  destination column for this arm
- `internal/transform/parquet_converter.go:339-411` — converter cannot preserve
  a field that does not exist in either output schema
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.1.0/xdr/xdr_generated.go:59683-59711`
  — upstream XDR arm carries `LedgerMaxDependentTxClusters`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.1.0/xdr/xdr_generated.go:61969-61979`
  — `GetContractParallelCompute()` exposes the arm to callers

## Evidence

`TransformConfigSetting()` explicitly reads older Soroban arms such as
`ContractExecutionLanes`, `ContractLedgerCostExt`, and
`LiveSorobanStateSizeWindow`, but there is no corresponding read path for
`ContractParallelCompute`. The current output schemas stop at the older
config-setting columns, so even a correct transform read would have nowhere to
store `LedgerMaxDependentTxClusters`.

## Anti-Evidence

Sparse-union config-setting rows are intentional for already-modeled arms, so a
metadata-only row is not automatically a bug by itself. The issue here is that
the entire live union arm is absent from both transform and schema coverage,
which means the exporter cannot represent the configured value at all.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced the complete config setting export path from XDR union definition through `TransformConfigSetting()` to both output schemas. The upstream XDR defines `ConfigSettingContractParallelComputeV0` (config_setting_id=14) with a single field `LedgerMaxDependentTxClusters` of type `Uint32`. The getter `GetContractParallelCompute()` is available on `ConfigSettingEntry`. However, `TransformConfigSetting()` in `config_setting.go` reads 12 other config setting arms (ids 0-12 and 15) but never calls `GetContractParallelCompute()`. Neither `ConfigSettingOutput` nor `ConfigSettingOutputParquet` defines a destination field. The result is a row with valid metadata (`config_setting_id=14`, `last_modified_ledger`, `closed_at`, etc.) but zero values for all payload columns — an empty shell that looks plausible but carries no useful data.

### Code Paths Examined

- `xdr_generated.go:59683-59684` — `ConfigSettingContractParallelComputeV0` struct with `LedgerMaxDependentTxClusters Uint32`
- `xdr_generated.go:61278` — `ConfigSettingIdConfigSettingContractParallelComputeV0 = 14`
- `xdr_generated.go:61969-61979` — `GetContractParallelCompute()` getter available on `ConfigSettingEntry`
- `internal/transform/config_setting.go:13-179` — `TransformConfigSetting()` reads arms 0-12 and 15 via their respective getters; no call to `GetContractParallelCompute()` exists
- `internal/transform/schema.go:563-627` — `ConfigSettingOutput` has no `LedgerMaxDependentTxClusters` field
- `internal/transform/schema_parquet.go:305-369` — `ConfigSettingOutputParquet` also lacks the field

### Findings

The gap is complete: transform, JSON schema, and Parquet schema all lack coverage for the `ContractParallelCompute` arm. When a ledger contains a `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` entry, the ETL emits a row with `config_setting_id=14` and valid metadata columns (`last_modified_ledger`, `ledger_entry_change`, `closed_at`, `ledger_sequence`) but every payload column is zero. Downstream BigQuery consumers querying `WHERE config_setting_id = 14` receive a row that appears valid but contains no representation of `ledger_max_dependent_tx_clusters`, making it impossible to track the protocol's parallel execution cluster limit.

This is not a sparse-union design issue (which was investigated and correctly rejected in fail/data-integrity/001). The sparse-union design intentionally zeros columns from *other* arms. Here, the *active* arm's own field has no destination column at all — the entire arm is unmodeled.

### PoC Guidance

- **Test file**: `internal/transform/config_setting_test.go`
- **Setup**: Create a mock `ingest.Change` with `ConfigSettingEntry` whose `ConfigSettingId` is `ConfigSettingIdConfigSettingContractParallelComputeV0` (14) and `ContractParallelCompute` set to `ConfigSettingContractParallelComputeV0{LedgerMaxDependentTxClusters: 42}`. Follow the existing test pattern (e.g., the `CONFIG_SETTING_CONTRACT_MAX_SIZE_BYTES` test case).
- **Steps**: Call `TransformConfigSetting(change, header)` with the mock change.
- **Assertion**: The returned `ConfigSettingOutput` should contain the value `42` in a `LedgerMaxDependentTxClusters` field. Currently, no such field exists, so the test will fail to compile — demonstrating the schema gap. Alternatively, assert that `config_setting_id == 14` and then show that no output field carries the value `42`, proving data loss.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestConfigParallelComputeEmptyShell"
**Test Language**: Go

### Demonstration

The test constructs a `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` entry (config_setting_id=14) with `LedgerMaxDependentTxClusters=42`, passes it through the production `TransformConfigSetting()` function, and verifies the output. The transform succeeds and returns `config_setting_id=14` with valid metadata (`last_modified_ledger=50000000`, `ledger_sequence=10`), but no output field carries the input value `42`. The entire JSON output was scanned field-by-field — every payload column is zero. This proves the active XDR union arm's only meaningful field is silently dropped, producing a plausible but hollow row.

### Test Body

```go
func TestConfigParallelComputeEmptyShell(t *testing.T) {
	const inputClusterLimit xdr.Uint32 = 42

	parallelCompute := xdr.ConfigSettingContractParallelComputeV0{
		LedgerMaxDependentTxClusters: inputClusterLimit,
	}

	ledgerEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 50000000,
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

	output, err := TransformConfigSetting(change, header)
	if err != nil {
		t.Fatalf("TransformConfigSetting returned unexpected error: %v", err)
	}

	// The transform correctly identifies this as config_setting_id=14.
	if output.ConfigSettingId != 14 {
		t.Fatalf("expected ConfigSettingId=14, got %d", output.ConfigSettingId)
	}

	// Serialize the output to JSON and search for the value 42.
	jsonBytes, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("failed to marshal output: %v", err)
	}

	jsonStr := string(jsonBytes)

	// Prove no output field carries the input value 42.
	foundValue := false
	var outputMap map[string]interface{}
	if err := json.Unmarshal(jsonBytes, &outputMap); err != nil {
		t.Fatalf("failed to unmarshal output: %v", err)
	}
	for key, val := range outputMap {
		switch v := val.(type) {
		case float64:
			if v == 42 {
				foundValue = true
				t.Logf("Found value 42 in field %q", key)
			}
		}
	}

	if !foundValue {
		t.Errorf("DATA LOSS CONFIRMED: CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0 (id=14) "+
			"with LedgerMaxDependentTxClusters=%d produces an empty-shell row. "+
			"No output field carries the input value. JSON output: %s",
			inputClusterLimit, jsonStr)
	}

	// Also verify metadata is present (the row looks plausible but is hollow).
	if output.LastModifiedLedger != 50000000 {
		t.Errorf("expected LastModifiedLedger=50000000, got %d", output.LastModifiedLedger)
	}
	if output.LedgerSequence != 10 {
		t.Errorf("expected LedgerSequence=10, got %d", output.LedgerSequence)
	}

	// Show the schema gap explicitly.
	zeroOutput := ConfigSettingOutput{}
	zeroOutput.ConfigSettingId = 14
	zeroOutput.LastModifiedLedger = output.LastModifiedLedger
	zeroOutput.LedgerEntryChange = output.LedgerEntryChange
	zeroOutput.ClosedAt = output.ClosedAt
	zeroOutput.LedgerSequence = output.LedgerSequence

	if output.ContractMaxSizeBytes != zeroOutput.ContractMaxSizeBytes &&
		output.LedgerMaxInstructions != zeroOutput.LedgerMaxInstructions {
		t.Logf("Some payload fields are non-zero — hypothesis may need revision")
	} else {
		fmt.Printf("CONFIRMED: config_setting_id=14 row has zero-valued payload columns "+
			"(ContractMaxSizeBytes=%d, LedgerMaxInstructions=%d) — "+
			"indistinguishable from an empty struct\n",
			output.ContractMaxSizeBytes, output.LedgerMaxInstructions)
	}
}
```

### Test Output

```
=== RUN   TestConfigParallelComputeEmptyShell
    data_integrity_poc_test.go:200: DATA LOSS CONFIRMED: CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0 (id=14) with LedgerMaxDependentTxClusters=42 produces an empty-shell row. No output field carries the input value. JSON output: {"config_setting_id":14,"contract_max_size_bytes":0,"ledger_max_instructions":0,"tx_max_instructions":0,"fee_rate_per_instructions_increment":0,"tx_memory_limit":0,"ledger_max_read_ledger_entries":0,"ledger_max_disk_read_entries":0,"ledger_max_read_bytes":0,"ledger_max_disk_read_bytes":0,"ledger_max_write_ledger_entries":0,"ledger_max_write_bytes":0,"tx_max_read_ledger_entries":0,"tx_max_disk_read_entries":0,"tx_max_read_bytes":0,"tx_max_disk_read_bytes":0,"tx_max_write_ledger_entries":0,"tx_max_write_bytes":0,"fee_read_ledger_entry":0,"fee_disk_read_ledger_entry":0,"fee_write_ledger_entry":0,"fee_read_1kb":0,"fee_write_1kb":0,"fee_disk_read_1kb":0,"bucket_list_target_size_bytes":0,"soroban_state_target_size_bytes":0,"write_fee_1kb_bucket_list_low":0,"rent_fee_1kb_soroban_state_size_low":0,"write_fee_1kb_bucket_list_high":0,"rent_fee_1kb_soroban_state_size_high":0,"bucket_list_write_fee_growth_factor":0,"soroban_state_rent_fee_growth_factor":0,"fee_historical_1kb":0,"tx_max_contract_events_size_bytes":0,"fee_contract_events_1kb":0,"ledger_max_txs_size_bytes":0,"tx_max_size_bytes":0,"fee_tx_size_1kb":0,"contract_cost_params_cpu_insns":[],"contract_cost_params_mem_bytes":[],"contract_data_key_size_bytes":0,"contract_data_entry_size_bytes":0,"max_entry_ttl":0,"min_temporary_ttl":0,"min_persistent_ttl":0,"auto_bump_ledgers":0,"persistent_rent_rate_denominator":0,"temp_rent_rate_denominator":0,"max_entries_to_archive":0,"bucket_list_size_window_sample_size":0,"live_soroban_state_size_window_sample_size":0,"live_soroban_state_size_window_sample_period":0,"eviction_scan_size":0,"starting_eviction_scan_level":0,"ledger_max_tx_count":0,"bucket_list_size_window":[],"live_soroban_state_size_window":[],"last_modified_ledger":50000000,"ledger_entry_change":0,"deleted":false,"closed_at":"1970-01-01T00:16:40Z","ledger_sequence":10}
CONFIRMED: config_setting_id=14 row has zero-valued payload columns (ContractMaxSizeBytes=0, LedgerMaxInstructions=0) — indistinguishable from an empty struct
--- FAIL: TestConfigParallelComputeEmptyShell (0.00s)
FAIL
```
