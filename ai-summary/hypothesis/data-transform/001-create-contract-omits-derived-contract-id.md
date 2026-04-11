# H001: `create_contract` details drop the deterministically derivable contract ID

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For successful `invoke_host_function` rows whose `details.type` is `create_contract` or `create_contract_v2`, `details.contract_id` should contain the contract ID implied by the operation's `ContractIdPreimage` and network ID. The output should let downstream analytics join the creation operation to the resulting contract rows and later contract events without re-reading external ledger state.

## Mechanism

Both creation branches call `contractIdFromTxEnvelope(transactionEnvelope)`, which only scans Soroban footprint `ContractData` keys and returns the first contract ID it finds. Fresh contract-creation envelopes do not carry the newly created contract instance in the footprint, so the helper returns `""` and the exporter writes a blank `contract_id` even though the XDR preimage is already present on the operation and is sufficient to derive the exact ID.

## Trigger

Export any successful Soroban `create_contract` or `create_contract_v2` transaction, especially one whose footprint does not already include the newly created contract instance key. The emitted `history_operations.details.contract_id` will be empty instead of the created contract's ID.

## Target Code

- `internal/transform/operation.go:1099-1108` - `create_contract` stores `contract_id` from `contractIdFromTxEnvelope(...)`
- `internal/transform/operation.go:1119-1136` - `create_contract_v2` repeats the same footprint-based lookup
- `internal/transform/operation.go:1808-1824` - helper only scans footprint `ContractData` keys
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:32277-32320` - `HashIdPreimageContractId` carries the exact derivation inputs
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/asset.go:451-469` - SDK already derives asset-based contract IDs from preimage + network

## Evidence

`operation_test.go` currently expects `contract_id: ""` for both `create_contract` and `create_contract_v2` rows (`internal/transform/operation_test.go:1943-1963`, `1999-2038`), confirming the live export shape is blank. The XDR model separately shows that contract IDs are defined as a hash over `HashIdPreimageContractId`, so the exporter already has enough information in the operation body to compute the missing value.

## Anti-Evidence

The repository's current tests intentionally accept blank `contract_id`, so this may be an unexamined historical contract rather than an accidental regression. A reviewer could also argue that the operation table only meant to expose preimage ingredients, but that is undercut by the presence of an explicit `contract_id` field that is currently populated with an unusable empty string.
