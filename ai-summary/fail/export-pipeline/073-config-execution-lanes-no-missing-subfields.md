# H073: `contract_execution_lanes` rows do not actually hide extra subfields

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the current `CONFIG_SETTING_CONTRACT_EXECUTION_LANES` XDR arm carried multiple independent execution-lane controls, the exported `config_settings` row should surface all of them instead of flattening the arm down to a single `ledger_max_tx_count` field.

## Mechanism

`TransformConfigSetting()` only reads `contractExecutionLanes.LedgerMaxTxCount`, which initially looked like a stale partial mapping for a newer multi-field config arm. If the arm had gained additional knobs upstream, the exporter would silently underfill every `contract_execution_lanes` row while still emitting a plausible-looking JSON object.

## Trigger

Export config-setting changes for any ledger containing a `CONFIG_SETTING_CONTRACT_EXECUTION_LANES` entry and compare the exported row against the current generated XDR definition for that union arm.

## Target Code

- `internal/transform/config_setting.go:TransformConfigSetting:98-100,170-172` — only `LedgerMaxTxCount` is read and exported
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:59506-59521` — current `ConfigSettingContractExecutionLanesV0` struct definition

## Evidence

The local transform names the arm `contractExecutionLanes` but exports only one scalar, which is a common sign of an older mapper lagging a richer protocol struct.

## Anti-Evidence

The current generated XDR struct for `ConfigSettingContractExecutionLanesV0` contains exactly one field: `LedgerMaxTxCount`. There are no hidden sibling fields for the exporter to miss.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected missing subfields do not exist in the current protocol definition. `TransformConfigSetting()` fully covers the present `ConfigSettingContractExecutionLanesV0` payload.

### Lesson Learned

Rich-sounding config-arm names are not evidence of partial export coverage by themselves. Before filing a "missing subfield" hypothesis, inspect the current generated XDR struct and confirm that the arm actually contains more fields than the transform exports.
