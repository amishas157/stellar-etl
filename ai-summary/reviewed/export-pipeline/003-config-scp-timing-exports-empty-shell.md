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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I verified that `ConfigSettingIdConfigSettingScpTiming` (ID=16) exists in the XDR SDK at `xdr/xdr_generated.go:61280` with a corresponding `ConfigSettingScpTiming` struct containing five `Uint32` timing fields: `LedgerTargetCloseTimeMilliseconds`, `NominationTimeoutInitialMilliseconds`, `NominationTimeoutIncrementMilliseconds`, `BallotTimeoutInitialMilliseconds`, and `BallotTimeoutIncrementMilliseconds`. The getter `GetContractScpTiming()` is defined at line 62021. The transformer in `config_setting.go` calls getters for 13 other union arms (IDs 0–12 plus the V0 cost extension) but never calls `GetContractScpTiming()`. The caller in `export_ledger_entry_changes.go:245-257` invokes `TransformConfigSetting` for every `LedgerEntryTypeConfigSetting` change without filtering by ID, so SCP timing entries are transformed but produce all-zero data rows.

### Code Paths Examined

- `internal/transform/config_setting.go:13-179` — `TransformConfigSetting()` calls getters for arms 0-12 + LedgerCostExt, discarding the `ok` boolean each time. No call to `GetContractScpTiming()` exists anywhere in the function. For ID=16, every getter returns `(zero-value, false)`, producing an output row with `config_setting_id=16` and all other fields zero.
- `internal/transform/schema.go:563-627` — `ConfigSettingOutput` has no fields for any of the five SCP timing values. There is nowhere to store the timing data even if the getter were called.
- `internal/transform/schema_parquet.go:305-369` — Parquet schema mirrors the JSON schema; also has no SCP timing fields.
- `cmd/export_ledger_entry_changes.go:245-257` — Calls `TransformConfigSetting` for all `LedgerEntryTypeConfigSetting` changes regardless of config setting ID. No pre-filter excludes unhandled arms.
- XDR SDK `xdr/xdr_generated.go:61051-61058` — `ConfigSettingScpTiming` struct with 5 timing fields confirmed.
- XDR SDK `xdr/xdr_generated.go:62021-62027` — `GetContractScpTiming()` getter confirmed to exist and return `(ConfigSettingScpTiming, bool)`.

### Findings

The bug is confirmed. When a `CONFIG_SETTING_SCP_TIMING` ledger entry (ID=16) is processed:

1. `configSetting.ConfigSettingId` correctly captures the value 16.
2. All 13 other getter calls (lines 26-107) return zero-value structs because the active union arm is `ContractScpTiming`, not any of the arms being queried. The discarded `ok=false` booleans are never checked.
3. `GetContractScpTiming()` is never called, so the actual timing data is never read.
4. The output struct has no fields for timing values, so even if the data were read, there would be nowhere to put it.
5. The resulting exported row has `config_setting_id=16` with all other numeric fields at zero, which is indistinguishable from "all timing values are zero" — a misleading silent data loss.

This same gap also affects `ConfigSettingIdConfigSettingEvictionIterator` (ID=13) and `ConfigSettingIdConfigSettingContractParallelComputeV0` (ID=14), which similarly have XDR getters (`GetEvictionIterator()`, `GetContractParallelComputeV0()`) that the transformer never calls and schema fields that don't exist.

### PoC Guidance

- **Test file**: `internal/transform/config_setting_test.go`
- **Setup**: Create a `ConfigSettingEntry` with `ConfigSettingId = ConfigSettingIdConfigSettingScpTiming` and populate `ContractScpTiming` with non-zero values (e.g., `LedgerTargetCloseTimeMilliseconds = 5000`). Wrap in an `ingest.Change` with a valid `LedgerHeaderHistoryEntry`.
- **Steps**: Call `TransformConfigSetting(change, header)` and inspect the returned `ConfigSettingOutput`.
- **Assertion**: Assert that `ConfigSettingId == 16` and demonstrate that no field in the output contains the timing value 5000 — all numeric fields will be zero. This proves that timing data is silently dropped. Additionally verify the output has no error (the function succeeds, masking the data loss).
