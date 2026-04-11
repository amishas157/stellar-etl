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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full path from `TransformContractData` (contract_data.go:49) through `ContractBalanceFromContractData` (contract_data.go:306-379) and compared it to the upstream SDK's identical function in `ingest/sac/contract_data.go:203`. Both implementations use `scAddress.GetContractId()` to extract the holder, which only succeeds for the `ScAddressTypeScAddressTypeContract` variant. This is by design because SAC (Stellar Asset Contract) balance `ContractData` entries only exist for contract-backed holders — account-backed holders' balances are stored in classic trustlines, not in Soroban storage.

### Code Paths Examined

- `internal/transform/contract_data.go:96-101` — calls `ContractBalanceFromContractData`, populates `balance_holder`/`balance` only when `dataBalance != nil`; encodes holder with `VersionByteContract`
- `internal/transform/contract_data.go:306-379` — local `ContractBalanceFromContractData` implementation; line 340 uses `scAddress.GetContractId()`, rejecting non-contract addresses
- `go-stellar-sdk/ingest/sac/contract_data.go:203-280` — upstream SDK's identical function also uses `scAddress.GetContractId()` on the same code path
- `go-stellar-sdk/ingest/sac/contract_data.go:512-560` — `BalanceToContractData` (inverse/test helper) explicitly constructs only `ScAddressTypeScAddressTypeContract` holders, confirming only contract holders exist in SAC ContractData
- `go-stellar-sdk/ingest/sac/contract_data.go:528-548` — `ContractBalanceLedgerKey` constructs the ledger key with `ScAddressTypeScAddressTypeContract` only
- `go-stellar-sdk/ingest/tutorial/expiring-sac-balances/main.go:61-80` — SDK tutorial also encodes holder with `VersionByteContract`, confirming design intent

### Why It Failed

The hypothesis is based on an incorrect assumption about the Stellar Asset Contract (SAC) storage model. In the SAC protocol:

1. **Account-backed balances** are stored in **classic trustlines** (for issued assets) or the account's native balance field (for XLM). They are NOT stored in Soroban `ContractData` entries.
2. **Contract-backed balances** are stored in Soroban `ContractData` entries with key `Balance(Address(ContractId))` because contracts cannot have classic trustlines.

Therefore, a SAC `ContractData` balance entry with key `Balance(Address(AccountId))` does not occur in normal protocol operation. The `GetContractId()` call is correct because the only SAC balance holders that appear in `ContractData` are contracts. The upstream SDK implements identical behavior, and its test helpers (`BalanceToContractData`, `ContractBalanceLedgerKey`) exclusively construct contract-addressed holders — confirming this is working-as-designed, not an oversight.

### Lesson Learned

The Stellar Asset Contract bridges classic and Soroban storage: account-backed balances live in classic trustlines while contract-backed balances live in Soroban `ContractData`. When evaluating ContractData balance extraction, always consider which address types actually appear in ContractData entries rather than assuming all `ScAddress` variants are equally likely. The upstream SDK's inverse/helper functions are useful for confirming design intent.
