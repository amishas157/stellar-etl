# H003: Native SAC contract instances export unsupported asset metadata

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformContractData()` should leave `asset_type`, `asset_code`, and `asset_issuer` empty for the native asset contract instance, matching the upstream SAC parser's explicit lumens exclusion. Native SAC rows should not be exported as if they were supported asset-metadata rows when the parser otherwise treats lumens as a special unsupported case.

## Mechanism

The local `AssetFromContractData()` fork diverged from the upstream SDK helper: upstream rejects the native asset contract ID before parsing asset info, while the ETL's local helper special-cases `"Native"` and returns `xdr.MustNewNativeAsset()`. `TransformContractData()` then copies that parsed asset into the exported row, yielding `asset_type="native"` with blank code and issuer for native SAC contract-instance rows.

## Trigger

Export a persistent `ContractData` ledger entry whose key is `ScvLedgerKeyContractInstance` and whose contract ID is the network native asset contract. The ETL will emit a row with `asset_type="native"`, while the upstream `ingest/sac.AssetFromContractData()` helper would reject the same row and leave the asset fields blank.

## Target Code

- `internal/transform/contract_data.go:84-91` — `TransformContractData()` copies parsed asset metadata into exported `asset_*` fields
- `internal/transform/contract_data.go:191-247` — local `AssetFromContractData()` accepts `"Native"` and returns `MustNewNativeAsset()`
- `go-stellar-sdk/ingest/sac/contract_data.go:74-94` — upstream helper rejects native asset contract IDs entirely
- `internal/transform/contract_data_test.go:142-150` — current local test expectation explicitly asserts `ContractDataAssetType: "native"`

## Evidence

The local helper has an ETL-only branch for `"Native"` that upstream removed in favor of an early lumens exclusion. The transform immediately trusts the helper result and serializes it into `asset_type`, so native SAC instance rows produce a partially-populated asset tuple that the upstream SAC parser would never emit.

## Anti-Evidence

The repository's current unit test already expects `ContractDataAssetType: "native"`, so maintainers may have intentionally accepted this divergence from upstream. If the project deliberately wants native SAC instances to surface as contract-data asset metadata, the behavior may be treated as a local contract rather than a bug.
