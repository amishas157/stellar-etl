# H001: Contract-data balance rows drop account-backed token holders

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a Soroban token balance `ContractDataEntry` is keyed by `DataKey::Balance(Address(AccountId))`, the transform should export the holder's account address in `balance_holder` and the stored token amount in `balance`. The output should preserve the holder address variant instead of only working for contract-backed holders.

## Mechanism

`ContractBalanceFromContractData()` only accepts `ScAddress.GetContractId()`, so valid balance rows keyed by an account address are rejected before the amount is returned. `TransformContractData()` also assumes every returned holder should be encoded with `VersionByteContract`, so the current path cannot faithfully represent account-backed balance holders and instead silently emits empty `balance_holder` and `balance` fields for those rows.

## Trigger

Process a ledger containing a SEP-41 / SAC token balance entry whose storage key is `Balance(Address(AccountId))`, which is the normal shape for a token balance owned by a Stellar account rather than another contract.

## Target Code

- `internal/transform/contract_data.go:96-100` — encodes any detected holder as a contract strkey and only fills `balance_holder`/`balance` when the helper succeeds
- `internal/transform/contract_data.go:321-343` — balance extraction rejects every `ScAddress` arm except `ContractId`

## Evidence

The helper reads the balance key as an `ScAddress`, but immediately narrows it with `scAddress.GetContractId()`. That matches contract-owned balances only. Because the caller receives only a raw 32-byte holder and not the address variant, it always uses `strkey.VersionByteContract` when populating `balance_holder`.

## Anti-Evidence

If the upstream token contract only ever stored balances under contract addresses, this path would be harmless. But Soroban token balance keys are defined as `Balance(Address)`, and `ScAddress` is a multi-arm union, so account-owned balances are a concrete valid trigger rather than a theoretical one.
