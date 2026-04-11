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
