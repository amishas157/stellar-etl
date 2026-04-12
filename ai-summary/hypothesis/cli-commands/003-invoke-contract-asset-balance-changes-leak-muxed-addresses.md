# H003: Invoke-contract balance-change details mix raw muxed and canonical account identities

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: operation-detail identity corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `export_operations` rows with `details.type="invoke_contract"`, the nested `details.asset_balance_changes[*].from` / `to` account fields should use the same identity convention as the rest of the ETL's account surfaces: canonical `G...` account IDs for the base field, with muxed identity preserved separately if needed. Downstream consumers should not see the same Stellar account represented as `G...` in one export surface and `M...` in another column that semantically means "account".

## Mechanism

`extractOperationDetails()` populates `asset_balance_changes` via `parseAssetBalanceChangesFromContractEvents()`, which passes the raw strings from `contractevents.NewStellarAssetContractEvent()` straight into `createSACBalanceChangeEntry()`. The SDK obtains those strings from `ScAddress.String()`, which returns `M...` for muxed accounts, and this helper never canonicalizes or splits that identity. As a result, invoke-contract balance-change details can export raw muxed addresses in `from` / `to` while sibling operation/effect helpers normalize account fields to canonical `G...` plus muxed companion metadata.

## Trigger

Run `export_operations` on a ledger containing an `invoke_contract` operation whose SAC transfer/mint/burn/clawback event references a muxed account. Inspect `details.asset_balance_changes`: the affected entry will carry `from` or `to` as an `M...` address instead of the canonical account form used elsewhere in operation exports.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:1063-1097` — wires `invoke_contract` details to `parseAssetBalanceChangesFromContractEvents()`
- `internal/transform/operation.go:parseAssetBalanceChangesFromContractEvents:1942-1975` — feeds raw SAC endpoint strings into the exported detail list
- `internal/transform/operation.go:createSACBalanceChangeEntry:1984-1998` — copies `from` / `to` directly without canonicalization or muxed metadata
- `github.com/stellar/go-stellar-sdk@v0.1.0/support/contractevents/utils.go:15-51` — source of the raw `from` / `to` strings
- `github.com/stellar/go-stellar-sdk@v0.1.0/xdr/scval.go:13-34` — muxed `ScAddress` stringification behavior

## Evidence

The local comment in `parseAssetBalanceChangesFromContractEvents()` says the parser expresses endpoints as account `G...` or contract `C...` strings, but the SDK now has an explicit muxed-account `ScAddress` arm and returns `M...` for it. Elsewhere in the transform layer, muxed-aware helpers like `addAccountAndMuxedAccountDetails()` and `addMuxed()` deliberately separate canonical and muxed identity, making this raw-string pass-through an outlier.

## Anti-Evidence

`asset_balance_changes` is a free-form JSON detail blob, not a strongly typed top-level schema, so one could argue raw `M...` values are still "exact" rather than strictly wrong. The hypothesis depends on the export contract favoring canonical account columns across surfaces, which is strongly suggested by sibling helpers but not spelled out for this nested field.
