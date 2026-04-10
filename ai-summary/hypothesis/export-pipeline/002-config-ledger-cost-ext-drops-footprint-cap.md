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
