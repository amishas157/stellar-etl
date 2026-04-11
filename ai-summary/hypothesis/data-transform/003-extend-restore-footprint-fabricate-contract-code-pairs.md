# H003: `extend_footprint_ttl` and `restore_footprint` fabricate contract/code pairs

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`extend_footprint_ttl` and `restore_footprint` operate on an arbitrary Soroban footprint, so their exported details should either identify a real `contract_id` / `contract_code_hash` pair from the same contract or avoid claiming a single pair when the footprint spans multiple contracts. The row should not synthesize a contract ID from one ledger key and a code hash from a different contract.

## Mechanism

These operation-details builders call `contractIdFromTxEnvelope()` and `contractCodeHashFromTxEnvelope()` independently. The first helper scans read-write keys before read-only keys and returns the first `ContractData` contract address; the second scans read-only keys before read-write keys and returns the first `ContractCode` hash. On mixed footprints, the helpers can select unrelated contracts from opposite halves of the footprint and combine them into one impossible pair.

## Trigger

Export an `extend_footprint_ttl` or `restore_footprint` transaction whose read-write footprint contains a `ContractData` key for contract A while the read-only footprint contains a `ContractCode` key for contract B. The exported operation-details row will show `contract_id = A` together with `contract_code_hash = hash(B)`, even though no single ledger entry in the footprint represents that association.

## Target Code

- `internal/transform/operation.go:1144-1159` — `extend_footprint_ttl` and `restore_footprint` attach a single `contract_id` and `contract_code_hash` to the row
- `internal/transform/operation.go:1808-1823` — `contractIdFromTxEnvelope()` picks the first `ContractData` key from read-write, then read-only
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope()` picks the first `ContractCode` key from read-only, then read-write
- `internal/transform/operation.go:1859-1873` — the exporter already emits `ledger_key_hash`, showing the operation can span multiple footprint entries

## Evidence

The two helpers intentionally search different key kinds with opposite precedence, yet their outputs are presented as one coherent contract/code identity in the final JSON. That makes the mismatch deterministic whenever the first `ContractData` and first `ContractCode` keys belong to different contracts, which is plausible for multi-contract restore/extend operations.

## Anti-Evidence

Single-contract footprints, or footprints where the first `ContractData` and first `ContractCode` keys happen to refer to the same contract, will hide the problem. There is also a chance reviewers decide these fields are only heuristic metadata for multi-key operations, but the current schema presents them as concrete values rather than best-effort hints.
