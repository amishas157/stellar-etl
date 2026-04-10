# H002: Transaction `tx_signers` exports signature bytes as fake account IDs

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_transactions.tx_signers` should contain the actual signing accounts or public keys for the transaction, or remain empty if the exporter cannot reliably recover them. Each element should correspond to a real signer identity, not to an encoded signature blob.

## Mechanism

`getTxSigners()` loops over `[]xdr.DecoratedSignature` but calls `strkey.Encode(strkey.VersionByteAccountID, sig.Signature)`. `sig.Signature` is the 64-byte signature payload, not the signer’s 32-byte public key, so the function emits valid-looking `G...` strings derived from signature bytes and `TransformTransaction()` stores them in `tx_signers` for both classic and fee-bump envelopes.

## Trigger

Export any transaction with one or more signatures. Compare `tx_signers` to the transaction’s real signer accounts: the emitted strings are computed from the raw signatures and do not identify the signing accounts.

## Target Code

- `internal/transform/transaction.go:getTxSigners:349-360` — encodes `sig.Signature` as an account ID
- `internal/transform/transaction.go:TransformTransaction:227-230` — fills `TxSigners` on the classic path
- `internal/transform/transaction.go:TransformTransaction:294-300` — fills `TxSigners` on the fee-bump path
- `internal/transform/schema.go:TransactionOutput:41-84` — labels the output field as `tx_signers`

## Evidence

The `xdr.DecoratedSignature` type contains only a 4-byte hint plus the signature bytes, but `getTxSigners()` never consults any signer key material. The helper directly encodes `sig.Signature` as an `AccountID`, and `transaction_test.go` hard-codes those synthetic `G...` strings as expected output, which means the mislabeled values currently ship as normal transaction data.

## Anti-Evidence

Recovering exact signer identities from decorated signatures alone may require more than the current helper has available, so the eventual fix could be a schema change rather than a code-only fix. Even so, the current exporter is still emitting wrong values under a field name that claims they are signers.
