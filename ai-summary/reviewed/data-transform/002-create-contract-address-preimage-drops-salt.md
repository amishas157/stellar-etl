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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `switchContractIdPreimageType()` (operation.go:2275-2294) which handles `ContractIdPreimageFromAddress` by extracting only `details["from"]` and `details["address"]`, completely ignoring `fromAddress.Salt`. The XDR struct `ContractIdPreimageFromAddress` contains both `Address ScAddress` and `Salt Uint256`. Critically, the canonical Horizon operations processor (`stellar/go services/horizon/internal/ingest/processors/operations_processor.go`) explicitly includes `details["salt"] = fromAddress.Salt.String()` for both `create_contract` and `create_contract_v2`, confirming this is not an intentional omission but a divergence from the authoritative Horizon output.

### Code Paths Examined

- `internal/transform/operation.go:2275-2294` — `switchContractIdPreimageType()` handles `FROM_ADDRESS` case: extracts `Address` but drops `Salt`
- `internal/transform/operation.go:1099-1113` — `create_contract` branch calls `switchContractIdPreimageType` and merges returned map into details
- `internal/transform/operation.go:1119-1140` — `create_contract_v2` branch does the same
- `internal/transform/operation_test.go:1943-1963` — test expectations confirm no `salt` key in output
- `stellar/go services/horizon/internal/ingest/processors/operations_processor.go` — Horizon inlines the same logic but DOES include `details["salt"] = fromAddress.Salt.String()` for both operation types
- `stellar/go processors/operation/operation.go` — intermediate `PreImageDetails` struct also omits `Salt`, but this helper is NOT the authoritative Horizon output path; the operations_processor.go inline code is
- `stellar/go-stellar-sdk xdr/xdr_generated.go` — `ContractIdPreimageFromAddress` struct confirms `Salt Uint256` is a first-class field

### Findings

The ETL's `switchContractIdPreimageType()` faithfully mirrors the SDK's `switchContractIdPreimage()` helper, which also omits salt. However, the canonical Horizon operations processor does NOT use that helper — it inlines the logic and explicitly adds `details["salt"]`. The ETL chose the wrong upstream reference, resulting in a structural data omission.

The salt is a 256-bit value that, combined with the deployer address and network passphrase, deterministically derives the contract ID. Without it, downstream consumers cannot reconstruct the contract ID derivation path from the exported preimage fields alone.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or a new `internal/transform/data_integrity_poc_test.go`)
- **Setup**: Construct a `LedgerTransaction` with an `InvokeHostFunction` operation of type `HostFunctionTypeHostFunctionTypeCreateContract`, using a `ContractIdPreimage` of type `FROM_ADDRESS` with a non-zero `Salt` value (e.g., `xdr.Uint256{0x01, 0x02, ...}`)
- **Steps**: Call `TransformOperation()` on the constructed transaction and inspect the returned `OperationOutput.OperationDetails` map
- **Assertion**: Assert that `details["salt"]` exists and equals the hex-encoded salt value. Currently it will be absent, proving the omission. The fix is to add `details["salt"] = fromAddress.Salt.String()` in the `FROM_ADDRESS` case of `switchContractIdPreimageType()`.
