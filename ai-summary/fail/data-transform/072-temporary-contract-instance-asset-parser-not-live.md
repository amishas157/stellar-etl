# H072: Temporary contract-instance asset parser drift is probably dead code

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If temporary `ContractData` rows can legally use `ScvLedgerKeyContractInstance`, the ETL should reject them before exporting SAC asset metadata, matching the upstream parser's persistent-durability precondition. Temporary contract-instance rows should not be interpreted as canonical asset-info rows.

## Mechanism

The local `AssetFromContractData()` helper lost the upstream `contractData.Durability != Persistent` guard, so a temporary contract-instance row with matching asset-info storage initially looked like it could export `asset_*` fields that upstream would suppress. That would create a durability-dependent divergence in `TransformContractData()` output.

## Trigger

Construct or ingest a `ContractData` ledger entry whose key type is `ScvLedgerKeyContractInstance`, whose durability is `ContractDataDurabilityTemporary`, and whose instance storage contains SAC `AssetInfo`. Run it through `TransformContractData()` and compare the result to the upstream SAC helper.

## Target Code

- `internal/transform/contract_data.go:191-202` — local `AssetFromContractData()` has no persistent-durability guard
- `go-stellar-sdk/ingest/sac/contract_data.go:79-84` — upstream helper rejects non-persistent contract-instance rows
- `go-stellar-sdk/txnbuild/invoke_host_function.go:114-123` — SDK constructs contract-instance ledger keys with `Durability: Persistent`

## Evidence

The local helper checks the key type but never checks durability before reading contract-instance storage. Upstream added the persistent-durability guard, so the ETL and SDK helpers do differ on paper for temporary contract-instance rows.

## Anti-Evidence

Every concrete SDK path I found that constructs a contract-instance ledger key uses `ContractDataDurabilityPersistent`, and Soroban instance storage is documented and modeled as persistent. I could not establish a legitimate on-chain path that produces a temporary `ScvLedgerKeyContractInstance` row, so the missing guard appears unreachable in practice.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not find a realistic trigger for a temporary contract-instance ledger entry. The SDK constructs contract-instance keys as persistent, and the upstream durability guard appears to be defensive hardening around a state shape that Soroban instance storage does not actually produce.

### Lesson Learned

A missing parser guard is only a live ETL bug if the protocol can produce the guarded shape. For Soroban contract-instance storage, verify the durability model first; otherwise an upstream/local drift may be dead code rather than export corruption.
