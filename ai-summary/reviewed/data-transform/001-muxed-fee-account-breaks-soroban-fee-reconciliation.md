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

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete Soroban fee reconciliation path in `TransformTransaction()`. At lines 154 and 158, `feeAccountAddress` is set via `MuxedAccount.Address()`, which the SDK confirms returns `M...` strkeys for muxed accounts (SDK `muxed_account.go:136-147`). The downstream `getAccountBalanceFromLedgerEntryChanges()` at line 318 compares this against `AccountId.Address()`, which the SDK confirms always returns `G...` strkeys (`account_id.go:28-35`). The string mismatch means both balance lookups return zero, producing `inclusionFeeCharged = -resourceFee` and `resourceFeeRefund = 0`. The same codebase already normalizes muxed addresses correctly at line 30 (`GetAccountAddressFromMuxedAccount`) and line 286 (`.ToAccountId()`), proving this is an oversight in the fee path.

### Code Paths Examined

- `internal/transform/transaction.go:154` — `feeAccountAddress = sourceAccount.Address()` returns `M...` for muxed source accounts
- `internal/transform/transaction.go:157-158` — `feeAccountAddress = feeBumpAccount.Address()` returns `M...` for muxed fee-bump accounts
- `internal/transform/transaction.go:177` — passes `feeAccountAddress` to `getAccountBalanceFromLedgerEntryChanges` for initial fee charge lookup
- `internal/transform/transaction.go:183,201` — passes same `feeAccountAddress` for resource fee refund lookup (V3 and V4 meta paths)
- `internal/transform/transaction.go:306-333` — `getAccountBalanceFromLedgerEntryChanges` compares `accountEntry.AccountId.Address()` (always `G...`) against `sourceAccountAddress` (could be `M...`)
- `go-stellar-sdk/xdr/muxed_account.go:136-147` — SDK `GetAddress()` encodes with `VersionByteMuxedAccount` for muxed type, producing `M...` strkey
- `go-stellar-sdk/xdr/account_id.go:28-35` — SDK `GetAddress()` encodes with `VersionByteAccountID`, always producing `G...` strkey
- `internal/utils/main.go:50-53` — `GetAccountAddressFromMuxedAccount()` correctly normalizes via `ToAccountId()`, producing `G...`
- `internal/transform/transaction.go:30` — `outputAccount` correctly uses `GetAccountAddressFromMuxedAccount()` (proof the codebase knows the pattern)
- `internal/transform/transaction.go:285-286` — Fee-bump detail section correctly uses `.ToAccountId()` (further proof of the oversight)

### Findings

The bug is confirmed at all three call sites where `feeAccountAddress` is used for balance lookups:
1. **Line 177** (FeeChanges lookup): `initialFeeCharged = 0 - 0 = 0`, so `outputInclusionFeeCharged = 0 - resourceFee = -resourceFee`. This produces a **negative inclusion fee** — a physically impossible value that will corrupt any downstream fee analysis.
2. **Line 183** (V3 TxChangesAfter lookup): `outputResourceFeeRefund = 0 - 0 = 0`. Any actual refund is lost.
3. **Line 201** (V4 TxChangesAfter + PostTxApplyFeeChanges lookup): Same zero result, refund lost.

The fix is to normalize `feeAccountAddress` at lines 154 and 158 by using either `GetAccountAddressFromMuxedAccount()` or `sourceAccount.ToAccountId().Address()` / `feeBumpAccount.ToAccountId().Address()`.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go`
- **Setup**: Create a `LedgerTransaction` with a muxed source account (`CryptoKeyTypeKeyTypeMuxedEd25519`) and valid Soroban data. Populate `FeeChanges` with ledger entry changes containing the underlying `G...` account ID with non-zero balance delta. Set `sorobanData.ResourceFee` to a known value (e.g., 38233).
- **Steps**: Call `TransformTransaction(transaction, lhe)` with the muxed-source Soroban transaction.
- **Assertion**: Assert that `result.InclusionFeeCharged` equals the expected positive value (initial fee charged minus resource fee), NOT the negative value `-38233`. Assert `result.ResourceFeeRefund` equals the expected refund amount, NOT zero. Also test the fee-bump variant by constructing a `FeeBump` envelope with a muxed `FeeBumpAccount`.
