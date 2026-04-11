# H003: Native SAC instance rows export unsupported lumens asset metadata

**Date**: 2026-04-11
**Subsystem**: data-integrity (transform)
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-contract-data` encounters the native-asset contract's persistent instance row, it should leave `contract_data_asset_type`, `contract_data_asset_code`, and `contract_data_asset_issuer` empty. The reference SAC parser explicitly treats lumens asset stats as unsupported and does not synthesize a native asset descriptor from contract storage.

## Mechanism

`AssetFromContractData()` special-cases `sym == "Native"` and returns `xdr.MustNewNativeAsset()` whenever the contract ID matches the native asset contract. `TransformContractData()` then extracts and exports `contract_data_asset_type = "native"` from that synthetic asset, even though the upstream helper rejects native-asset contract rows entirely and the rest of the local contract-data balance path does not provide a consistent lumens asset-stat model.

## Trigger

Export contract-data changes across a ledger containing the native asset contract's persistent contract-instance row with `AssetInfo = ["Native", ...]`.

## Target Code

- `internal/transform/contract_data.go:84-90` — `TransformContractData()` copies any parsed asset into the `contract_data_asset_*` columns.
- `internal/transform/contract_data.go:191-247` — `AssetFromContractData()` recognizes `"Native"` and returns a native asset for the native contract ID.

## Evidence

The local helper explicitly branches on `case "Native"` at `contract_data.go:243-247` and returns a native asset object. The upstream implementation at `go-stellar-sdk/ingest/sac/contract_data.go:90-94` rejects the same contract ID with the comment `we don't support asset stats for lumens`.

## Anti-Evidence

This may be an intentional local extension if stellar-etl wants a partial native-asset view that upstream SAC code omits. But there is no evidence of a fully supported lumens stats contract in the surrounding transform code, which makes the one-off native asset export path look more like an accidental fork drift than a documented schema choice.
