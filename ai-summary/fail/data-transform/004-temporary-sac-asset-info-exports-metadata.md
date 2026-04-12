# H001: Temporary SAC asset-info rows export verified asset metadata

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformContractData()` should only populate `contract_data_asset_type`, `contract_data_asset_code`, and `contract_data_asset_issuer` for persistent Stellar Asset Contract instance rows. If a `ContractData` row has `contract_durability = ContractDataDurabilityTemporary`, those asset columns should remain empty because the upstream SAC parser rejects non-persistent asset-info entries before parsing any metadata.

## Mechanism

The local `AssetFromContractData()` fork in `internal/transform/contract_data.go` dropped the upstream `contractData.Durability != xdr.ContractDataDurabilityPersistent` guard. `TransformContractData()` then unconditionally copies any returned asset into the exported `contract_data_asset_*` columns. A temporary row built from a real SAC asset-info payload therefore exports a self-contradictory record: `contract_durability` says the row is temporary, but the asset columns still present it as the verified canonical metadata entry for that asset contract.

## Trigger

1. Build a real SAC asset-info ledger entry with the upstream helper `sac.AssetToContractData(false, "USDC", <issuer>, <contractID>)` for any AlphaNum4 or AlphaNum12 asset.
2. Flip the resulting `ContractDataEntry.Durability` from `Persistent` to `Temporary`.
3. Feed the row through `TransformContractData()` with the production `AssetFromContractData`.
4. Observe that the exported row keeps `contract_durability = ContractDataDurabilityTemporary` but still fills `contract_data_asset_type`, `contract_data_asset_code`, and `contract_data_asset_issuer`, while the upstream `sac.AssetFromContractData()` rejects the same row.

## Target Code

- `internal/transform/contract_data.go:84-91` — `TransformContractData()` copies any parsed asset into `contract_data_asset_*`
- `internal/transform/contract_data.go:191-297` — local `AssetFromContractData()` omits the durability guard
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/sac/contract_data.go:74-84` — upstream helper rejects non-persistent rows before parsing
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/sac/contract_data.go:459-494` — upstream test helper constructs a real SAC asset-info row with persistent durability

## Evidence

The upstream SAC helper now returns early unless `contractData.Durability == Persistent`. The local fork still checks only `contractData.Key.Type == ScvLedgerKeyContractInstance`, then parses instance storage and verifies the contract ID against the reconstructed asset contract ID. That means a temporary row built from a real SAC asset-info payload is still accepted locally and exported as a verified asset descriptor.

## Anti-Evidence

The parser still has strong anti-forgery checks: it requires contract-instance storage, the canonical `AssetInfo` storage slot, a valid AlphaNum4/AlphaNum12 asset payload, and a recomputed contract ID match. The dead `"Native"` branch is not part of this trigger because the local length guard still rejects the 1-element native encoding.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the local `AssetFromContractData()` (lines 191–297) and confirmed it lacks the upstream durability guard at line 82 of the SDK's `sac/contract_data.go`. However, the local function's first gate at line 196 requires `contractData.Key.Type == ScvLedgerKeyContractInstance`. By Soroban protocol design, contract instance entries (`ScvLedgerKeyContractInstance` key type) are always created with `ContractDataDurabilityPersistent` — the protocol enforces this invariant at the ledger level. A contract data entry with instance key type and temporary durability cannot exist on the real Stellar network.

### Code Paths Examined

- `internal/transform/contract_data.go:191-198` — local `AssetFromContractData()` checks `contractData.Key.Type != ScvLedgerKeyContractInstance` but not durability
- `internal/transform/contract_data.go:84-91` — `TransformContractData()` unconditionally copies parsed asset into output fields
- `go-stellar-sdk/.../sac/contract_data.go:74-84` — upstream adds durability guard as defense-in-depth
- `go-stellar-sdk/.../xdr/xdr_generated.go:55260` — `ScValTypeScvLedgerKeyContractInstance` is key type 20, used only for the singleton contract instance entry

### Why It Failed

The trigger requires a `ContractDataEntry` with `Key.Type == ScvLedgerKeyContractInstance` AND `Durability == Temporary`. This combination is impossible on the real Stellar network — the Soroban protocol enforces that contract instance entries are always persistent. The trigger explicitly requires artificially "flipping" durability on a constructed entry, which would never appear in actual ledger data processed by the ETL. The missing durability guard is a defense-in-depth omission, not a data correctness bug that produces wrong output on real data.

### Lesson Learned

When a hypothesis requires artificially constructing a protocol-invalid ledger entry to trigger, verify whether that entry state can actually occur on the real network. Missing defense-in-depth guards are code quality issues but do not produce data corruption if the protocol invariant they guard is always enforced at the ledger level.
