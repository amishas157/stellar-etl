# H002: `invoke_contract` can report another contract's `contract_code_hash`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For an `invoke_contract` operation, the exported `details.contract_code_hash` should correspond to the same contract identified by `details.contract_id`. If the operation invokes contract A, the row should not attribute contract B's code hash to contract A just because B also appears in the Soroban footprint.

## Mechanism

The `invoke_contract` branch takes `contract_id` directly from `HostFunction.MustInvokeContract().ContractAddress`, which identifies the actual target contract. But `contract_code_hash` comes from `contractCodeHashFromTxEnvelope()`, a helper that scans the footprint for the first `ContractCode` ledger key in read-only order and then read-write order, without checking whether that code entry belongs to the invoked contract. A multi-contract footprint can therefore export `contract_id = A` together with `contract_code_hash = hash(B)`.

## Trigger

Export a Soroban transaction whose `invoke_contract` operation targets contract A but whose footprint also contains at least one other `LedgerKeyContractCode` entry for contract B before A's code key in the helper's scan order. The resulting operation-details JSON should show contract A's address with contract B's code hash.

## Target Code

- `internal/transform/operation.go:1075-1087` — `invoke_contract` details use the explicit contract address plus `contractCodeHashFromTxEnvelope()`
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope()` returns the first code key from the footprint without matching it to `contract_id`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:56389-56452` — `ContractExecutable` encodes the canonical Wasm hash identity for a contract instance

## Evidence

The code path uses two independent sources for fields that are presented as one logical pair: `contract_id` comes from the host function target, while `contract_code_hash` is chosen by a generic "first code key wins" footprint scan. Nothing in the helper constrains that hash to the invoked contract, so any dependency/library code entry that appears earlier in the footprint can be misattributed to the target contract.

## Anti-Evidence

If the footprint contains only one contract-code key, or if the invoked contract's code happens to appear first in the scan order, the exported row will look correct. Current tests use simple fixtures and do not exercise multi-contract footprints with competing code keys.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the `invoke_contract` branch in `extractOperationDetails` (line 1075) which sets `contract_id` from `invokeArgs.ContractAddress.String()` — a precise, operation-specific source. In contrast, `contract_code_hash` is set via `contractCodeHashFromTxEnvelope()` (line 1841), which iterates all footprint ledger keys in ReadOnly-then-ReadWrite order and returns the Wasm hash of the first `ContractCode` entry found. The function `contractCodeFromContractData` (line 1876) performs only a type check (`GetContractCode`), with no filtering by contract identity. In a multi-contract footprint, whichever Wasm code entry appears first wins, regardless of which contract was actually invoked.

### Code Paths Examined

- `internal/transform/operation.go:1075-1085` — `invoke_contract` branch: `contract_id` from `invokeArgs.ContractAddress`, `contract_code_hash` from `contractCodeHashFromTxEnvelope()`
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope()`: iterates ReadOnly then ReadWrite footprint, returns first `ContractCode` hash found
- `internal/transform/operation.go:1876-1884` — `contractCodeFromContractData()`: checks `GetContractCode()` and returns `Hash.HexString()`, no contract identity filtering
- `internal/transform/operation.go:1808-1823` — `contractIdFromTxEnvelope()`: same "first match" pattern for `ContractData` keys, also affected for `create_contract`/`extend_footprint_ttl`/`restore_footprint` operations
- `internal/transform/operation.go:1695-1707` — Duplicate code path in `transactionOperationWrapper.Details()` with the same bug

### Findings

1. **Root cause confirmed**: `contractCodeHashFromTxEnvelope` is a transaction-level scan that returns the first code hash from the footprint, but `contract_id` (for `invoke_contract`) is operation-level. These two fields are presented as a logical pair but can disagree when multiple contracts with different Wasm code appear in the footprint.

2. **Scope is broader than the hypothesis states**: The function is called from 12 different sites across `invoke_contract`, `create_contract`, `create_contract_v2`, `upload_wasm`, `extend_footprint_ttl`, and `restore_footprint` operations — all in both `extractOperationDetails` and `transactionOperationWrapper.Details()`. All share the same "first code key wins" behavior.

3. **`contractIdFromTxEnvelope` has the same structural issue**: It also does a "first `ContractData` key wins" scan (line 1808). For operations like `extend_footprint_ttl` and `restore_footprint`, both `contract_id` and `contract_code_hash` come from independent first-match scans of different key types, meaning they may reference different contracts.

4. **Multi-contract footprints are realistic**: Any `invoke_contract` where the invoked contract calls another contract (cross-contract call) will have footprint entries for both. Library/utility contracts (token contracts, oracles) frequently appear in footprints alongside the primary invoked contract.

5. **The bug is silent**: No error is raised; the output looks structurally valid. Downstream consumers (analytics, indexers) would associate the wrong Wasm code hash with a contract, corrupting code-level attribution.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Construct a `LedgerTransaction` with an `InvokeHostFunction` operation of type `invoke_contract` targeting contract A. Build a `SorobanTransactionData` footprint with two `ContractCode` entries in ReadOnly: first for Wasm hash B, then for Wasm hash A. Set `HostFunction.InvokeContract.ContractAddress` to contract A's address.
- **Steps**: Call `TransformOperation()` (or `extractOperationDetails`) on this transaction.
- **Assertion**: Assert that `details["contract_id"]` equals contract A's address AND `details["contract_code_hash"]` equals Wasm hash B (not A) — demonstrating the mismatch. The correct behavior would be for `contract_code_hash` to equal Wasm hash A, matching the invoked contract.
