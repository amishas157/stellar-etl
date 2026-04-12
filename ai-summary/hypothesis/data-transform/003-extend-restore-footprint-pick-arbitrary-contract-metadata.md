# H003: `extend_footprint_ttl` and `restore_footprint` export arbitrary first-match contract metadata

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `extend_footprint_ttl` and `restore_footprint`, exported `details.contract_id` and `details.contract_code_hash` should describe the actual target of the operation, or remain empty when the footprint spans multiple contracts/code entries and no single authoritative value exists. The transform should not fabricate a singular contract identifier by picking whichever matching footprint entry happens to appear first.

## Mechanism

Both operation cases populate `ledger_key_hash` with the full footprint set, then separately fill `contract_id` and `contract_code_hash` by scanning the entire Soroban footprint and returning the first matching `ContractData` or `ContractCode` key. These TTL operations apply to a footprint set, not a uniquely identified contract, so a legitimate footprint containing keys for multiple contracts or unrelated contract-code entries will export a plausible but wrong singular `contract_id` / `contract_code_hash` that depends on footprint ordering rather than the actual restored/extended target set.

## Trigger

Construct a Soroban `extend_footprint_ttl` or `restore_footprint` transaction whose footprint contains keys for contract A and contract B, or a contract-data key plus an unrelated contract-code key that sorts earlier in the helper scan order. Export the operation: `details.ledger_key_hash` will show the full set, but `details.contract_id` and/or `details.contract_code_hash` will name only the first matching footprint entry.

## Target Code

- `internal/transform/operation.go:1144-1159` — `extend_footprint_ttl` and `restore_footprint` assign singular contract metadata from footprint-wide helper scans.
- `internal/transform/operation.go:1808-1856` — `contractIdFromTxEnvelope()` and `contractCodeHashFromTxEnvelope()` return the first match found in the footprint.

## Evidence

The helpers do not inspect which ledger key the TTL operation is semantically about; they just scan `ReadWrite` / `ReadOnly` arrays and return on the first match. The same operation details already export `ledger_key_hash` as a list, which is strong evidence that the underlying target is a set of keys rather than one canonical contract identifier.

## Anti-Evidence

Single-contract footprints will look correct, and some downstream consumers may treat these singular fields as heuristics. But the current export still presents them as authoritative scalar metadata even when the footprint legitimately spans multiple contracts or code entries.
