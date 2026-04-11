# H004: `restore_footprint` mislabels SAC balance restores as the asset contract

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a `restore_footprint` operation restores a Stellar Asset Contract balance owned by another contract, `details.contract_id` should identify the holder contract being restored, or the exporter should avoid claiming a single contract ID at all. It should not attribute the row to the SAC token contract if the restored state belongs to a different contract.

## Mechanism

`restore_footprint` populates `details.contract_id` via `contractIdFromTxEnvelope()`, which reads the `Contract` field of the first footprint `ContractData` key. But `sac.ContractBalanceLedgerKey(assetContractId, holderID)` intentionally encodes contract balances under the **asset contract** as the outer `ContractData.Contract`, while the actual holder contract is nested inside the key vector. The helper therefore returns the asset contract ID and silently misattributes contract-balance restores.

## Trigger

Build or export a `restore_footprint` transaction that restores a SAC balance for a contract holder. The resulting `history_operations.details.contract_id` will point at the token's SAC contract instead of the contract whose balance key is being restored.

## Target Code

- `internal/transform/operation.go:1153-1159` - `restore_footprint` writes `contract_id` from the footprint helper
- `internal/transform/operation.go:1808-1838` - helper extracts only `LedgerKeyContractData.Contract.ContractId`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/txnbuild/restore_footprint.go:39-83` - SDK builds asset-balance restoration footprints with `sac.ContractBalanceLedgerKey(...)`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/sac/contract_data.go:524-549` - SAC balance key stores `assetContractId` as outer contract and holder contract inside the key vector

## Evidence

The footprint helper is purely structural: it never inspects the key payload, so any SAC balance key is reduced to its outer asset-contract ID. The upstream SDK's `ContractBalanceLedgerKey()` implementation makes that mismatch explicit by placing `holderID` in the vector component while `ContractData.Contract` is set to `assetContractId`.

## Anti-Evidence

This bug depends on a specific footprint shape: contract-balance restoration. For ordinary contract-instance footprints, the helper may still return the intended contract. A reviewer might also prefer that `contract_id` on restore operations describe the storage-owning contract (the SAC contract) rather than the restored holder, so the table's intended meaning needs confirmation.
