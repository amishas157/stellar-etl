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
