# H003: Contract-data export drops SAC balances when the holder is an account address

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Stellar Asset Contract balance rows, `TransformContractData()` should export the holder address and exact balance regardless of whether the `Balance(Address)` key points at an account or a contract. An account-held balance entry should therefore populate `balance_holder` with the holder's `G...` address (or another unambiguous holder identifier) and `balance` with the decoded amount string.

## Mechanism

`ContractBalanceFromContractData()` reads the holder from the `Balance(Address)` key but only accepts the `ScAddress` contract arm via `GetContractId()`. When the holder is a regular account address, the parser returns `(zero, nil, false)`, and `TransformContractData()` leaves both `ContractDataBalanceHolder` and `ContractDataBalance` empty even though the row is a valid SAC balance entry. Because most real token balances are account-held, the export silently strips the financial fields from a live class of contract-data rows.

## Trigger

1. Export a `LedgerEntryTypeContractData` row representing a Stellar Asset Contract balance keyed by an account holder rather than a contract holder.
2. Run `export_ledger_entry_changes --export-contract-data`.
3. Compare the row to the ledger entry: `balance_holder` and `balance` will be blank even though the storage key and value encode a real holder and amount.

## Target Code

- `internal/transform/contract_data.go:96-100` — the top-level transformer only populates balance fields when the parser returns a holder and amount
- `internal/transform/contract_data.go:321-378` — the parser requires `scAddress.GetContractId()` and rejects the account arm of `ScAddress`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:56740-56860` — `ScAddress` explicitly supports both `AccountId` and `ContractId`
- `internal/transform/contract_data_test.go:61-76` — tests inject mock balance parsers and never exercise the real account-vs-contract branch

## Evidence

The generated XDR union shows `ScAddress` has distinct `AccountId` and `ContractId` arms, but the balance parser only reads the contract arm. The `Balance(Address)` storage key is therefore decoded too narrowly: account-held balances fail the `GetContractId()` check before the code ever reaches the amount map. The surrounding transformer treats a nil amount as "not a balance row," so the exported row retains the contract metadata while silently losing the actual holder and balance.

## Anti-Evidence

Contract-held balances still export correctly, so fixtures built around contract-address holders or mocked balance parsers will pass. Non-balance contract-data rows are also supposed to leave these columns empty, which can mask the bug unless the reviewer compares against a real SAC balance entry keyed by an account.
