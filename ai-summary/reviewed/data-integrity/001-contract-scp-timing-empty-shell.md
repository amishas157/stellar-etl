# H001: `CONFIG_SETTING_CONTRACT_SCP_TIMING` exports as an empty shell row

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a config-setting ledger entry uses the live XDR arm `ContractScpTiming`, the exported `config_settings` row should preserve the five timing values carried on-chain: `ledger_target_close_time_milliseconds`, `nomination_timeout_initial_milliseconds`, `nomination_timeout_increment_milliseconds`, `ballot_timeout_initial_milliseconds`, and `ballot_timeout_increment_milliseconds`. JSON and Parquet should both expose those values instead of a metadata-only shell.

## Mechanism

`TransformConfigSetting()` reads many `ConfigSettingEntry` union arms, but it never calls `GetContractScpTiming()`. The JSON schema, Parquet schema, and `ToParquet()` converter also define no destination columns for the timing payload, so any `CONFIG_SETTING_CONTRACT_SCP_TIMING` entry is silently flattened into a plausible row that keeps only metadata like `config_setting_id`, `last_modified_ledger`, and `closed_at`.

## Trigger

Run `export_ledger_entry_changes --export-config-settings` on a ledger range containing a `LedgerEntryTypeConfigSetting` entry whose active union arm is `ContractScpTiming`.

## Target Code

- `internal/transform/config_setting.go:24-107` — reads many config-setting union arms but never reads `ContractScpTiming`
- `internal/transform/config_setting.go:116-178` — constructs the output row with no SCP-timing payload fields
- `internal/transform/schema.go:564-627` — JSON schema has no SCP-timing columns
- `internal/transform/schema_parquet.go:305-369` — Parquet schema also omits the timing fields
- `internal/transform/parquet_converter.go:339-410` — Parquet conversion can only copy the incomplete JSON schema
- `cmd/export_ledger_entry_changes.go:245-256` — exports every config-setting entry through `TransformConfigSetting()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/xdr_generated.go:61051-61057` — upstream XDR defines `ConfigSettingScpTiming` with five timing fields
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/xdr_generated.go:62019-62027` — upstream XDR exposes `GetContractScpTiming()`

## Evidence

The current transformer already has the same sparse-union pattern that produced the published `CONFIG_SETTING_CONTRACT_PARALLEL_COMPUTE_V0` and `CONFIG_SETTING_EVICTION_ITERATOR` empty-shell findings: it ignores a live union arm, while the export command still serializes the row. The upstream SDK now exposes a concrete `ContractScpTiming` arm with five scalar values, but there is no matching read path or schema field anywhere in `stellar-etl`.

## Anti-Evidence

This depends on ledgers actually containing the new `ContractScpTiming` arm; older protocol ranges will never trigger it. I have not produced a PoC here, so the exact enum value and first live ledger still need confirmation.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full path from `export_ledger_entry_changes.go:245` where all `LedgerEntryTypeConfigSetting` entries are routed unconditionally to `TransformConfigSetting()`, through the transformer at `config_setting.go:24-178` which reads 12 config-setting union arms but never calls `GetContractScpTiming()`, and confirmed that `ConfigSettingOutput` in `schema.go`, `ConfigSettingOutputParquet` in `schema_parquet.go`, and the Parquet converter all define zero destination fields for any of the five SCP timing values. The SDK version actually used by this project (`v0.0.0-20251211085638-ba09a6a91775`) defines `ConfigSettingScpTiming` at `xdr_generated.go:61051-61057` with enum value 16 and a working `GetContractScpTiming()` accessor at line 62019.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:245-256` — routes all `LedgerEntryTypeConfigSetting` changes to `TransformConfigSetting()` without filtering by arm; no guard prevents config_setting_id=16 from being processed
- `internal/transform/config_setting.go:24-107` — calls `GetContractMaxSizeBytes`, `GetContractCompute`, `GetContractLedgerCost`, `GetContractLedgerCostExt`, `GetContractHistoricalData`, `GetContractEvents`, `GetContractBandwidth`, `GetContractCostParamsCpuInsns`, `GetContractCostParamsMemBytes`, `GetContractDataKeySizeBytes`, `GetContractDataEntrySizeBytes`, `GetStateArchivalSettings`, `GetContractExecutionLanes`, `GetLiveSorobanStateSizeWindow` — but never `GetContractScpTiming()`
- `internal/transform/config_setting.go:116-178` — output struct construction has no fields for any SCP timing value
- `internal/transform/schema.go` — `ConfigSettingOutput` struct has no SCP timing fields (confirmed via grep for "scp", "timing", "ballot", "nomination")
- `internal/transform/schema_parquet.go` — `ConfigSettingOutputParquet` struct also lacks SCP timing fields
- `internal/transform/parquet_converter.go` — no SCP timing field mapping
- `go-stellar-sdk xdr_generated.go:61051-61057` — defines `ConfigSettingScpTiming{LedgerTargetCloseTimeMilliseconds, NominationTimeoutInitialMilliseconds, NominationTimeoutIncrementMilliseconds, BallotTimeoutInitialMilliseconds, BallotTimeoutIncrementMilliseconds}` (all `Uint32`)
- `go-stellar-sdk xdr_generated.go:61280` — `ConfigSettingIdConfigSettingScpTiming = 16`
- `go-stellar-sdk xdr_generated.go:62019-62029` — `GetContractScpTiming()` accessor exists and returns `ConfigSettingScpTiming`

### Findings

This is structurally identical to the confirmed findings 005 (ParallelCompute) and 006 (EvictionIterator). The `ConfigSettingEntry` union has arm `ContractScpTiming` (enum id=16) with 5 uint32 payload fields, but the transformer, JSON schema, Parquet schema, and Parquet converter all completely ignore this arm. When a ledger contains a `CONFIG_SETTING_SCP_TIMING` entry, the export pipeline emits a plausible-looking row with `config_setting_id=16` and correct metadata (`last_modified_ledger`, `closed_at`, `ledger_sequence`) but all 5 timing payload values are silently lost — they default to zero and are indistinguishable from genuine zero values.

### PoC Guidance

- **Test file**: `internal/transform/config_setting_test.go`
- **Setup**: Construct a mock `ingest.Change` with `LedgerEntryTypeConfigSetting` where `ConfigSettingId = ConfigSettingIdConfigSettingScpTiming` (16) and `ContractScpTiming` populated with non-zero values for all 5 fields (e.g., `LedgerTargetCloseTimeMilliseconds=5000`, `NominationTimeoutInitialMilliseconds=1000`, etc.)
- **Steps**: Call `TransformConfigSetting(change, header)` and inspect the returned `ConfigSettingOutput`
- **Assertion**: Assert that the output contains the 5 SCP timing field values. Currently this will fail because the schema has no fields for them, confirming the empty-shell bug. The PoC should demonstrate that the row is emitted (no error) but the timing payload is absent.
