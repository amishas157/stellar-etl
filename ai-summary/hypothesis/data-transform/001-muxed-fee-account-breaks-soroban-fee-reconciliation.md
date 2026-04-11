# H001: Muxed fee accounts break Soroban fee reconciliation

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Soroban transactions, `inclusion_fee_charged` and `resource_fee_refund` should be computed from the actual fee-paying account's balance changes even when the source account or fee-bump fee source is a muxed `M...` address. The transform should normalize muxed accounts to their underlying `G...` account before matching ledger-entry balance changes, so a valid muxed Soroban fee payer produces the same fee breakdown as the equivalent unmuxed account.

## Mechanism

`TransformTransaction()` stores `feeAccountAddress` using `MuxedAccount.Address()`, which returns an `M...` strkey for muxed accounts, but `getAccountBalanceFromLedgerEntryChanges()` compares that string against `AccountId.Address()` values from ledger entries, which are always unmuxed `G...` account IDs. When the fee payer is muxed, both balance lookups miss, so the function computes `initialFeeCharged = 0`, producing corrupt fee-breakdown fields such as negative `inclusion_fee_charged` and zero `resource_fee_refund`.

## Trigger

Process any protocol-20+ Soroban transaction whose fee-paying account is muxed:

1. A classic Soroban transaction with a muxed source account, or
2. A Soroban fee-bump transaction whose `FeeSource` is muxed.

If the transaction charges a resource fee or later refunds part of it, the exported row will reconcile those amounts against a missing balance delta. For example, a transaction with `resource_fee = 38233` and no matched fee-account balance change will export `inclusion_fee_charged = -38233`.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:148-159` — stores the Soroban fee payer as `sourceAccount.Address()` / `feeBumpAccount.Address()`
- `internal/transform/transaction.go:TransformTransaction:177-202` — computes `inclusion_fee_charged` and `resource_fee_refund` from matched balance changes
- `internal/transform/transaction.go:getAccountBalanceFromLedgerEntryChanges:306-333` — matches balance changes against unmuxed `AccountId.Address()` strings
- `internal/utils/main.go:GetAccountAddressFromMuxedAccount:49-54` — existing helper already converts muxed accounts to the underlying `G...` address
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/muxed_account.go:GetAddress:118-147` — SDK returns `M...` strkeys for `KEY_TYPE_MUXED_ED25519`

## Evidence

The transform already uses `GetAccountAddressFromMuxedAccount()` for the exported `account` field, showing the codebase knows muxed accounts must be normalized for account-level data. But the fee-reconciliation branch bypasses that helper and uses `MuxedAccount.Address()` directly. The downstream balance scan then compares that `M...` string to ledger-entry `AccountId.Address()` output, which can only be `G...`, so no balance change can match for a muxed fee payer.

## Anti-Evidence

Most existing transaction fixtures use unmuxed fee payers, so the happy path continues to produce correct values and existing tests pass. Non-Soroban transactions also avoid this branch entirely, so the corruption is limited to Soroban fee accounting rather than every transaction row.
