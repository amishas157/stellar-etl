# H066: Unknown contract-id preimage arm panic is future-only

**Date**: 2026-04-14
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For every currently valid create-contract operation, the exporter should serialize the contract-id preimage fields that exist on-chain and emit stable operation details for both supported preimage encodings.

## Mechanism

`switchContractIdPreimageType()` panics on any unrecognized `ContractIdPreimageType`, so a real unhandled preimage arm would abort the export instead of preserving contract-creation details. Because both create-contract code paths call this helper directly, the blast radius would cover normal operation exports and transaction-operation wrapper exports.

## Trigger

Process a `CREATE_CONTRACT` or `CREATE_CONTRACT_V2` host function whose `ContractIdPreimage.Type` is not one of the currently generated XDR enum values.

## Target Code

- `internal/transform/operation.go:1099-1113` — `create_contract` path calls `switchContractIdPreimageType()`
- `internal/transform/operation.go:1121-1140` — `create_contract_v2` path calls `switchContractIdPreimageType()`
- `internal/transform/operation.go:1721-1735` — wrapper path for `create_contract`
- `internal/transform/operation.go:1741-1762` — wrapper path for `create_contract_v2`
- `internal/transform/operation.go:2275-2294` — helper panics on unknown contract-id preimage types
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:28479-28505` — current `ContractIdPreimageType` enum only defines `FromAddress` and `FromAsset`

## Evidence

The helper has only two explicit cases and a panic default, so any current missing preimage arm would become a concrete export failure at the moment the ETL tried to format create-contract details.

## Anti-Evidence

The generated XDR enum currently defines only `CONTRACT_ID_PREIMAGE_FROM_ADDRESS` and `CONTRACT_ID_PREIMAGE_FROM_ASSET`, and the helper handles both. There is no legitimate current ledger encoding that reaches the panic path.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected gap depends on a future `ContractIdPreimageType` arm that does not exist in the current SDK/XDR, so it cannot corrupt today's exports.

### Lesson Learned

Helper panics around union subtypes need a concrete current enum gap before they count as live findings. A narrow helper with full coverage of the present generated XDR is a dead end for current-chain data-integrity work.
