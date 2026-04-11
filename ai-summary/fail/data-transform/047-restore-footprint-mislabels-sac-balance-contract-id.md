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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `contractIdFromTxEnvelope()` in `internal/transform/operation.go:1808-1838` and compared it against the canonical upstream Horizon SDK implementation in `stellar/go` at `ingest/ledger_transaction.go:840-893`. Both implementations use the identical extraction: `contractData.Contract.GetContractId()`, which returns the storage-owning contract from the `LedgerKeyContractData.Contract` field. The upstream SDK's `RestoreFootprintDetails()` explicitly calls `o.Transaction.ContractIdFromTxEnvelope()` with this same semantics.

### Code Paths Examined

- `internal/transform/operation.go:1153-1159` — `restore_footprint` case calls `contractIdFromTxEnvelope(transactionEnvelope)` to populate `details["contract_id"]`
- `internal/transform/operation.go:1808-1838` — `contractIdFromTxEnvelope` iterates ReadWrite then ReadOnly footprint keys, calling `contractIdFromContractData` which extracts `contractData.Contract.GetContractId()`
- `stellar/go ingest/ledger_transaction.go:840-893` — upstream SDK's `ContractIdFromTxEnvelope()` and `contractIdFromContractData()` use the identical extraction logic
- `stellar/go processors/operation/restore_footprint_details.go:12-36` — upstream `RestoreFootprintDetails()` populates `ContractID` from the same `ContractIdFromTxEnvelope()` method
- `stellar-go-sdk ingest/sac/contract_data.go:524-549` — `ContractBalanceLedgerKey()` sets `ContractData.Contract` to `assetContractId` by design; the holder is an internal key component

### Why It Failed

The `contract_id` field is designed to identify the **storage-owning contract** (the contract whose storage namespace the `ContractData` key belongs to). For SAC balance keys, the asset contract IS the storage owner — the holder contract is merely an internal key component within the asset contract's storage namespace. Both the stellar-etl and the canonical upstream Horizon SDK (`stellar/go`) use the identical extraction (`contractData.Contract.GetContractId()`), confirming this is the intended semantic. The hypothesis misunderstands the Soroban storage model and treats working-as-designed behavior as a bug.

### Lesson Learned

The `ContractData.Contract` field in Soroban identifies the contract that owns the storage namespace, not any entity referenced within the key payload. For SAC balance keys, the asset contract is the storage owner by protocol design. Always check the canonical upstream SDK implementation before claiming an ETL field is misattributed — if the upstream uses the same extraction, the behavior is ecosystem convention.
