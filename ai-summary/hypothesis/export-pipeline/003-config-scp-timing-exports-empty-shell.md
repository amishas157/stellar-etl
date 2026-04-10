# H003: `CONFIG_SETTING_SCP_TIMING` exports as a zero-filled shell row

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a config-setting row has `config_setting_id = CONFIG_SETTING_SCP_TIMING`, the export should surface the active timing parameters: `ledgerTargetCloseTimeMilliseconds`, `nominationTimeoutInitialMilliseconds`, `nominationTimeoutIncrementMilliseconds`, `ballotTimeoutInitialMilliseconds`, and `ballotTimeoutIncrementMilliseconds`. A consumer should be able to reconstruct the on-chain SCP timing config from the exported row.

## Mechanism

`TransformConfigSetting()` never calls `GetContractScpTiming()`, and the output schemas define no fields for any SCP timing values. Because the function also ignores the boolean return values from unrelated union-arm getters, a live `CONFIG_SETTING_SCP_TIMING` entry degrades into a row that only preserves `config_setting_id` and metadata while all exported numeric fields stay at unrelated zero defaults.

## Trigger

Run `export_ledger_entry_changes --export-config-settings` on ledgers that contain a `CONFIG_SETTING_SCP_TIMING` entry.

## Target Code

- `internal/transform/config_setting.go:24-104` — transformer reads many union arms but never reads `GetContractScpTiming()`
- `internal/transform/schema.go:563-627` — `ConfigSettingOutput` has no columns for any SCP timing values
- `internal/transform/schema_parquet.go:305-369` — Parquet schema also omits SCP timing fields
- `internal/transform/config_setting_test.go:14-59` — tests only cover an older arm and do not exercise newer timing entries

## Evidence

Current XDR includes a dedicated `ConfigSettingScpTiming` struct with five timing fields, but none of those names appear anywhere in the repo's transform or schema code. The transformer pattern here is to call getters for older arms, ignore the `ok` flag, and then serialize zero values into the shared output struct when the active arm is something newer and unhandled.

## Anti-Evidence

The export still preserves `config_setting_id`, so a downstream reader can tell which arm the zero-filled row represents. But there is no exported field that contains the actual timing parameters, so the row looks valid while omitting the configuration it claims to represent.
