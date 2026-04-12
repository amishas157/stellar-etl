# H001: SAC account balance rows are dropped from contract-data exports

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a Stellar Asset Contract balance entry is keyed by an account address, `soroban_contract_data.balance_holder` should export that holder's address and `soroban_contract_data.balance` should export the stored balance amount. Account-held SAC balances should not disappear just because the holder is a `G...`/`M...` address instead of a `C...` contract.

## Mechanism

`ContractBalanceFromContractData()` correctly recognizes that the storage key is `DataKey::Balance(Address)`, but after extracting the `ScAddress` it immediately calls `GetContractId()` and rejects every non-contract address arm. `TransformContractData()` then leaves `balance_holder` and `balance` empty for legitimate account-held SAC balance rows, silently turning a real token balance into an apparently non-balance contract-data row.

## Trigger

1. Export contract-data changes for a ledger containing a Stellar Asset Contract mint or transfer to a regular account.
2. Locate the `ContractDataEntry` whose key is `Balance(Address(account))`.
3. Observe that the exported row has empty `balance_holder` / `balance` even though the on-chain storage entry contains a positive amount for that account.

## Target Code

- `internal/transform/contract_data.go:96-100` — only populates `balance_holder` / `balance` when `ContractBalanceFromContractData()` returns a non-nil amount
- `internal/transform/contract_data.go:321-378` — parses `Balance(Address)` keys but accepts only `ScAddressTypeContract` via `GetContractId()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/scval.go:13-48` — `ScAddress.String()` supports account, contract, muxed-account, liquidity-pool, and claimable-balance arms
- `https://raw.githubusercontent.com/stellar/rs-soroban-env/da325551829d31dcbfa71427d51c18e71a121c5f/soroban-env-host/src/native_contract/token/storage_types.rs` — upstream SAC storage key is `DataKey::Balance(Address)`, not `Balance(ContractId)`

## Evidence

The upstream token storage definition explicitly says the balance key carries a generic `Address`. In local code, the parser extracts that address with `GetAddress()` but then narrows it to `GetContractId()`, so any `Address::Account` balance key is treated as non-balance data even though `xdr.ScAddress.String()` already provides the correct string rendering for account holders.

## Anti-Evidence

The checked-in `testdata/changes/contract_data.golden` currently shows only `C...` non-empty `balance_holder` values, so this path may be under-covered by fixtures today. If current ledgers happen to exercise mostly contract-held balances, the impact is less visible, but the parser still rejects the broader address domain that the protocol permits.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full code path from `TransformContractData` (line 96) through `ContractBalanceFromContractData` (line 306–378), confirming that `scAddress.GetContractId()` at line 340 rejects non-contract addresses. Then compared against the upstream SDK implementation at `go-stellar-sdk/ingest/sac/contract_data.go:203–280`, which contains the **identical** `GetContractId()` call. The upstream SDK intentionally only extracts contract-held SAC balances because account-held SAC balances are canonically represented by trustline ledger entries, not contract data entries. The local code is a faithful fork that correctly mirrors this design.

### Code Paths Examined

- `internal/transform/contract_data.go:96-101` — `TransformContractData` only populates `balance_holder`/`balance` when the helper returns non-nil; encodes the holder with `VersionByteContract` (C... prefix), confirming it's designed only for contract-type holders
- `internal/transform/contract_data.go:306-378` — Local `ContractBalanceFromContractData` calls `scAddress.GetContractId()` at line 340, rejecting account-type addresses
- `go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/ingest/sac/contract_data.go:203-280` — Upstream SDK has the identical `GetContractId()` call at line ~243, confirming this is intentional design, not a local fork bug
- `go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/xdr_generated.go:56834-56843` — `GetContractId()` returns ok=true only for `ScAddressTypeScContract` arm

### Why It Failed

The behavior is **by design in the upstream Stellar SDK**, not a local fork bug. The `ContractBalanceFromContractData` function in both the upstream SDK and the local fork intentionally only extracts **contract-held** SAC balances (C... addresses). Account-held SAC balances (G... addresses) are excluded because:

1. **Account balances are canonical in trustlines**: When a SAC contract processes a transfer to a regular Stellar account, the Soroban host automatically creates/updates a trustline for that account. The trustline is the canonical balance record.
2. **Contract data is redundant for accounts**: The contract data balance entry for an account is an implementation detail of SAC storage, not the canonical source of truth.
3. **Return type enforces this**: The function returns `[32]byte` representing a `ContractId` (Hash), and line 99 of `TransformContractData` encodes it with `VersionByteContract`. An account public key encoded with the contract version byte would produce a nonsensical address.
4. **The raw data is still exported**: The contract data row itself (key, val, XDR) is still exported — only the parsed `balance_holder`/`balance` convenience fields are empty, which is correct.

### Lesson Learned

When a local helper is a fork of an upstream SDK function, always check the upstream implementation before hypothesizing that the local behavior is a bug. If both implementations have the same logic, the behavior is by design. For SAC balance extraction specifically, `GetContractId()` is intentional because account balances are canonically tracked via trustlines, not contract data entries.
