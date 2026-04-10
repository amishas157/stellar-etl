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
