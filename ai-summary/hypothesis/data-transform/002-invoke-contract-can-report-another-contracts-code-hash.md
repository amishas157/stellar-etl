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
