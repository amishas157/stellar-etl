# 019: `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` exports as an empty shell row

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformConfigSetting()` silently drops the only payload field from the live XDR arm `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0`. When `export_ledger_entry_changes --export-config-settings` encounters that ledger entry, it still emits a plausible row with `config_setting_id=14`, but `ledger_max_dependent_tx_clusters` never survives into JSON or Parquet output.

## Root Cause

The transformer reads many older `ConfigSettingEntry` union arms, but it never calls `GetContractParallelCompute()`. `ConfigSettingOutput`, `ConfigSettingOutputParquet`, and `ConfigSettingOutput.ToParquet()` also define no destination column for `ledger_max_dependent_tx_clusters`, so the value is lost before serialization and the exporter emits a metadata-only shell row.

## Reproduction

When a normal ledger-entry-changes export processes a `LedgerEntryTypeConfigSetting` row whose `ConfigSettingId` is `ConfigSettingIdConfigSettingContractParallelComputeV0`, the command unconditionally calls `TransformConfigSetting()` and appends its result to the `config_settings` output. The resulting row preserves metadata such as `config_setting_id`, `ledger_sequence`, and `closed_at`, but the source XDR value `ledger_max_dependent_tx_clusters` is absent from every exported field and from the Parquet schema.

## Affected Code

- `internal/transform/config_setting.go:13-179` — transforms config-setting union arms but never reads `GetContractParallelCompute()`
- `internal/transform/schema.go:563-627` — JSON schema omits `ledger_max_dependent_tx_clusters`
- `internal/transform/schema_parquet.go:305-369` — Parquet schema also omits `ledger_max_dependent_tx_clusters`
- `internal/transform/parquet_converter.go:339-410` — Parquet conversion can only copy the incomplete schema
- `cmd/export_ledger_entry_changes.go:245-257` — exports every config-setting entry through `TransformConfigSetting()` without filtering unsupported arms
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:59683-59685` — upstream XDR defines `ConfigSettingContractParallelComputeV0` with `LedgerMaxDependentTxClusters`
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:61969-61979` — upstream XDR exposes `GetContractParallelCompute()`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestConfigSettingParallelComputeDropped`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"encoding/json"
	"reflect"
	"strings"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestConfigSettingParallelComputeDropped(t *testing.T) {
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
		Post:       &ledgerEntry,
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{CloseTime: 1000},
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

	jsonBytes, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("json.Marshal() error = %v", err)
	}

	var jsonOutput map[string]any
	if err := json.Unmarshal(jsonBytes, &jsonOutput); err != nil {
		t.Fatalf("json.Unmarshal() error = %v", err)
	}

	if _, ok := jsonOutput["ledger_max_dependent_tx_clusters"]; ok {
		t.Fatal("JSON unexpectedly exported ledger_max_dependent_tx_clusters")
	}

	parquetOutput, ok := output.ToParquet().(ConfigSettingOutputParquet)
	if !ok {
		t.Fatalf("ToParquet() returned %T, want ConfigSettingOutputParquet", output.ToParquet())
	}

	parquetType := reflect.TypeOf(parquetOutput)
	for i := range parquetType.NumField() {
		field := parquetType.Field(i)
		if strings.Contains(field.Tag.Get("parquet"), "name=ledger_max_dependent_tx_clusters") {
			t.Fatal("Parquet schema unexpectedly exported ledger_max_dependent_tx_clusters")
		}
	}

	if output.LedgerMaxTxCount != 0 {
		t.Fatalf("LedgerMaxTxCount = %d, want 0 for unrelated arm", output.LedgerMaxTxCount)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: A `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` export row should preserve `ledger_max_dependent_tx_clusters` from the source XDR arm.
- **Actual**: The exporter emits a row with `config_setting_id=14`, but `ledger_max_dependent_tx_clusters` is missing from the transformed struct, the JSON output, and the Parquet schema.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC constructs the real `ConfigSettingContractParallelComputeV0` XDR arm, runs the production transformer, and shows the value has no JSON field and no Parquet destination.
2. Realistic preconditions: YES — `export_ledger_entry_changes` transforms every `LedgerEntryTypeConfigSetting` entry through this path in normal operation.
3. Bug vs by-design: BUG — the exporter does not reject or skip unsupported config-setting arms; it emits a normal-looking row that silently drops the only business value.
4. Final severity: High — this is structural corruption of exported network-configuration data, not a direct monetary miscalculation.
5. In scope: YES — it is a concrete production export path producing silently incomplete output.
6. Test correctness: CORRECT — the test uses the real XDR arm and production transform/parquet conversion paths, with a distinctive sentinel value and direct schema checks.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Add `ledger_max_dependent_tx_clusters` to `ConfigSettingOutput`, `ConfigSettingOutputParquet`, and `ConfigSettingOutput.ToParquet()`, populate it from `configSetting.GetContractParallelCompute()` inside `TransformConfigSetting()`, and extend config-setting tests to cover this union arm so future XDR additions cannot regress into metadata-only shell rows.
