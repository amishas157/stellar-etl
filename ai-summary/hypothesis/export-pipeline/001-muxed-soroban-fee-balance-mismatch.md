# H001: Muxed Soroban fee accounts break balance-delta fee exports

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Soroban transactions, `inclusion_fee_charged` and `resource_fee_refund` should be derived from the fee-paying account's actual balance deltas even when the source or fee account is muxed. A muxed `M...` account should still match the underlying ledger `AccountId`, so the exported fee fields should reflect the same non-negative values visible in the fee-account balance changes.

## Mechanism

`TransformTransaction()` uses `MuxedAccount.Address()` to build `feeAccountAddress`, which yields an `M...` string for muxed accounts. `getAccountBalanceFromLedgerEntryChanges()` only compares against `accountEntry.AccountId.Address()`, which is the canonical `G...` account ID stored in ledger-entry changes, so muxed fee payers never match and both start/end balances stay zero. The exported `inclusion_fee_charged` and `resource_fee_refund` then collapse to arithmetic on zero instead of the real balance deltas.

## Trigger

Run `export_transactions` on a Soroban transaction whose source account is muxed, or on a Soroban fee-bump transaction whose fee account is muxed, with a non-zero resource fee or refund.

## Target Code

- `internal/transform/transaction.go:151-159` — fee-account lookup key is taken from `sourceAccount.Address()` / `feeBumpAccount.Address()`
- `internal/transform/transaction.go:177-202` — exported fee fields are computed from `getAccountBalanceFromLedgerEntryChanges(...)`
- `internal/transform/transaction.go:306-333` — balance-delta scan only matches `accountEntry.AccountId.Address()`
- `internal/utils/main.go:49-53` — existing helper already canonicalizes muxed accounts to their underlying `AccountId`

## Evidence

The transformer already uses `GetAccountAddressFromMuxedAccount()` for the exported `account` field, which shows the codebase knows ledger state is keyed by canonical `G...` account IDs. But the Soroban fee-account path bypasses that helper and feeds raw muxed `M...` addresses into a matcher that only ever inspects `AccountId.Address()`. `go doc` for `xdr.MuxedAccount.Address` and `xdr.AccountId.Address` confirms these are different encodings for muxed vs canonical accounts.

## Anti-Evidence

Non-muxed source and fee accounts are unaffected because both sides use `G...` addresses in that case. The bug is therefore narrow to muxed-account Soroban transactions, which may make it easy to miss in ordinary fixtures.
