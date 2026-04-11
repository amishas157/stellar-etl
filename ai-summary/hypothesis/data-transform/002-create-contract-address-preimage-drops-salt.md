# H002: Address-based contract creation details discard the preimage salt

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `create_contract` and `create_contract_v2` operations whose `ContractIdPreimage` is `FROM_ADDRESS`, the exported details should include both the deployer address and the 256-bit `salt`. The salt is part of the on-chain preimage and is required to distinguish otherwise identical deployments and to recompute the resulting contract ID.

## Mechanism

`switchContractIdPreimageType()` handles `ContractIdPreimageFromAddress` by exporting only `"from": "address"` and `"address": ...`. The XDR struct explicitly contains both `Address` and `Salt`, so two deployments from the same address with different salts silently collapse to the same exported preimage payload whenever the other derived fields are equal or blank.

## Trigger

Export two successful `create_contract` or `create_contract_v2` operations from the same deployer address but with different salts. The resulting `history_operations.details` rows will contain the same address-only preimage payload instead of preserving the salt that differentiates the contracts.

## Target Code

- `internal/transform/operation.go:1108-1113` - `create_contract` copies the truncated preimage map into details
- `internal/transform/operation.go:1135-1140` - `create_contract_v2` uses the same helper
- `internal/transform/operation.go:2275-2294` - `switchContractIdPreimageType()` drops `Salt` for `FROM_ADDRESS`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:28560-28570` - `ContractIdPreimageFromAddress` contains both `Address` and `Salt`

## Evidence

The checked-in expectations for address-based `create_contract` rows include `from` and `address` but no salt at all (`internal/transform/operation_test.go:1943-1963`). The underlying XDR definition shows `Salt Uint256` is a first-class preimage member, so the omission is not a protocol limitation or an upstream abstraction gap.

## Anti-Evidence

Each operation row still has a unique operation ID, so the missing salt does not make rows literally duplicate at the table level. A reviewer could argue that the ETL only intended to emit a human-friendly preimage summary, but that leaves the details payload unable to reconstruct the actual contract ID inputs it claims to describe.
