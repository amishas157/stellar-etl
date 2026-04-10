# H002: `CONFIG_SETTING_EVICTION_ITERATOR` exports an empty shell

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-config-settings` encounters a
`CONFIG_SETTING_EVICTION_ITERATOR` ledger entry, the exported row should retain
the on-chain iterator state: `bucket_list_level`, `is_curr_bucket`, and
`bucket_file_offset`. JSON and Parquet consumers should see the real iterator
position instead of an all-default shell row.

## Mechanism

The current XDR union contains an `EvictionIterator` arm, but
`TransformConfigSetting()` never calls `GetEvictionIterator()`. Neither
`ConfigSettingOutput` nor `ConfigSettingOutputParquet` defines columns for the
iterator fields, so the exporter emits a plausible row with
`config_setting_id=13` and standard metadata while silently discarding all
iterator position data before serialization.

## Trigger

1. Run `export_ledger_entry_changes --export-config-settings` across a ledger
   batch that includes `CONFIG_SETTING_EVICTION_ITERATOR`.
2. Locate the emitted `config_settings` row with `config_setting_id=13`.
3. Compare it with the source XDR entry and observe that
   `bucket_list_level`, `is_curr_bucket`, and `bucket_file_offset` never appear
   in the exported JSON or Parquet row.

## Target Code

- `internal/transform/config_setting.go:13-179` — transform never reads
  `GetEvictionIterator()`
- `internal/transform/schema.go:563-627` — JSON schema lacks iterator columns
- `internal/transform/schema_parquet.go:305-369` — Parquet schema likewise has
  no iterator columns
- `internal/transform/parquet_converter.go:339-411` — converter only preserves
  fields already present in the sparse config-setting schema
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.1.0/xdr/xdr_generated.go:60965-61011`
  — upstream `EvictionIterator` arm carries three concrete fields
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.1.0/xdr/xdr_generated.go:61944-61955`
  — `GetEvictionIterator()` makes that arm available to callers

## Evidence

The transform handles adjacent config-setting arms introduced in nearby
protocol work (`ContractExecutionLanes`, `LiveSorobanStateSizeWindow`,
`ContractLedgerCostExt`) but never touches `EvictionIterator`. The output
schema has no `bucket_list_level`, `is_curr_bucket`, or `bucket_file_offset`
field, so even non-zero iterator state must collapse to a metadata-only row.

## Anti-Evidence

Some config-setting rows intentionally populate only the active union arm and
leave unrelated columns zeroed. That sparse-row design does not explain this
case, because the active arm itself has no representable destination columns in
the current exporter.
