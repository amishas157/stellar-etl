# 005: `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` exports as an empty shell row

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformConfigSetting()` silently drops the only payload field from the live XDR arm `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0`. When `export_ledger_entry_changes --export-config-settings` encounters that ledger entry, it still emits a plausible row with `config_setting_id=14`, but neither JSON nor Parquet output preserves `ledger_max_dependent_tx_clusters`.

## Root Cause

The transformer reads many `ConfigSettingEntry` union arms, but it never calls `GetContractParallelCompute()`. `ConfigSettingOutput`, `ConfigSettingOutputParquet`, and `ConfigSettingOutput.ToParquet()` also define no destination column for `ledger_max_dependent_tx_clusters`, so the value is lost before serialization and the exporter produces a metadata-only sparse-union shell.

## Reproduction

When a normal ledger-entry-changes export processes a `LedgerEntryTypeConfigSetting` row whose `ConfigSettingId` is `ConfigSettingIdConfigSettingContractParallelComputeV0`, `cmd/export_ledger_entry_changes.go` always routes it through `TransformConfigSetting()` and appends the result to the `config_settings` output. The source XDR arm contains `LedgerMaxDependentTxClusters`, but the exported row preserves only metadata such as `config_setting_id`, `last_modified_ledger`, `closed_at`, and `ledger_sequence`.

## Affected Code

- `internal/transform/config_setting.go:24-107` — reads many config-setting union arms but never calls `GetContractParallelCompute()`
- `internal/transform/config_setting.go:116-178` — constructs the output row from existing fields only, leaving no place for the parallel-compute payload
- `internal/transform/schema.go:563-627` — JSON schema omits `ledger_max_dependent_tx_clusters`
- `internal/transform/schema_parquet.go:305-369` — Parquet schema also omits the field
- `internal/transform/parquet_converter.go:339-410` — Parquet conversion can only copy the incomplete JSON schema
- `cmd/export_ledger_entry_changes.go:245-256` — exports every config-setting entry through `TransformConfigSetting()` without filtering out unsupported arms
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:59683-59684` — upstream XDR defines `ConfigSettingContractParallelComputeV0.LedgerMaxDependentTxClusters`
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:61969-61979` — upstream XDR exposes `GetContractParallelCompute()`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestConfigParallelComputeEmptyShell`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `internal/transform/data_integrity_poc_test.go`
  3. Run: `go test ./internal/transform/... -run TestConfigParallelComputeEmptyShell -v`
  4. Observe: JSON and Parquet output omit `ledger_max_dependent_tx_clusters` even though the source XDR arm contains `42`.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestConfigParallelComputeEmptyShell(t *testing.T) {
	const inputClusterLimit xdr.Uint32 = 42

	parallelCompute := xdr.ConfigSettingContractParallelComputeV0{
		LedgerMaxDependentTxClusters: inputClusterLimit,
	}

	configSetting := xdr.ConfigSettingEntry{
		ConfigSettingId:         xdr.ConfigSettingIdConfigSettingContractParallelComputeV0,
		ContractParallelCompute: &parallelCompute,
	}

	gotParallelCompute, ok := configSetting.GetContractParallelCompute()
	if !ok {
		t.Fatal("expected ConfigSettingEntry to expose ContractParallelCompute via XDR getter")
	}
	if gotParallelCompute != parallelCompute {
		t.Fatalf("GetContractParallelCompute() = %#v, want %#v", gotParallelCompute, parallelCompute)
	}

	ledgerEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 50000000,
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
			ScpValue:  xdr.StellarValue{CloseTime: 1000},
			LedgerSeq: 10,
		},
	}

	output, err := TransformConfigSetting(change, header)
	if err != nil {
		t.Fatalf("TransformConfigSetting() error = %v", err)
	}

	if output.ConfigSettingId != int32(xdr.ConfigSettingIdConfigSettingContractParallelComputeV0) {
		t.Fatalf("ConfigSettingId = %d, want %d", output.ConfigSettingId, xdr.ConfigSettingIdConfigSettingContractParallelComputeV0)
	}

	jsonPayload, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("json.Marshal(output) error = %v", err)
	}

	var jsonDecoded map[string]any
	if err := json.Unmarshal(jsonPayload, &jsonDecoded); err != nil {
		t.Fatalf("json.Unmarshal(output) error = %v", err)
	}

	parquetPayload, err := json.Marshal(output.ToParquet())
	if err != nil {
		t.Fatalf("json.Marshal(output.ToParquet()) error = %v", err)
	}

	var parquetDecoded map[string]any
	if err := json.Unmarshal(parquetPayload, &parquetDecoded); err != nil {
		t.Fatalf("json.Unmarshal(parquet) error = %v", err)
	}

	jsonValue, jsonOK := jsonDecoded["ledger_max_dependent_tx_clusters"]
	parquetValue, parquetOK := parquetDecoded["ledger_max_dependent_tx_clusters"]

	if !jsonOK || !parquetOK || jsonValue != float64(inputClusterLimit) || parquetValue != float64(inputClusterLimit) {
		t.Fatalf(
			"expected config_setting_id=%d to preserve ledger_max_dependent_tx_clusters=%d in JSON and Parquet; json_ok=%v json_value=%v parquet_ok=%v parquet_value=%v json=%s parquet=%s",
			xdr.ConfigSettingIdConfigSettingContractParallelComputeV0,
			inputClusterLimit,
			jsonOK,
			jsonValue,
			parquetOK,
			parquetValue,
			string(jsonPayload),
			string(parquetPayload),
		)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: A `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` export row should preserve `ledger_max_dependent_tx_clusters` from the source XDR arm in both JSON and Parquet output.
- **Actual**: The exporter emits a row with `config_setting_id=14`, but neither JSON nor Parquet contains `ledger_max_dependent_tx_clusters`; the only configured value is silently dropped.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC populates the real XDR arm, verifies the SDK getter returns `42`, then runs the production transformer and Parquet conversion and shows both outputs omit the field entirely.
2. Realistic preconditions: YES — `export_ledger_entry_changes` sends every config-setting ledger entry through `TransformConfigSetting()` in normal operation.
3. Bug vs by-design: BUG — unlike the legacy bucket-list aliases retained for backward compatibility, this arm has no old/new alias pair or any destination column at all; the exporter emits a live config-setting row while dropping its only payload.
4. Final severity: High — this is structural corruption of exported protocol-configuration data, not a direct monetary miscalculation.
5. In scope: YES — it is a concrete production code path that silently exports incomplete data.
6. Test correctness: CORRECT — the test uses the real getter, the real transformer, the real Parquet conversion path, and a distinctive sentinel value that is absent from metadata.
7. Alternative explanations: NONE — there is no existing JSON or Parquet field that aliases or preserves `ledger_max_dependent_tx_clusters`.
8. Novelty: NOVEL

## Suggested Fix

Add `LedgerMaxDependentTxClusters` to `ConfigSettingOutput`, `ConfigSettingOutputParquet`, and `ConfigSettingOutput.ToParquet()`, populate it from `configSetting.GetContractParallelCompute()` in `TransformConfigSetting()`, and extend config-setting tests so `config_setting_id=14` exports a populated sparse-union row instead of a metadata-only shell.
