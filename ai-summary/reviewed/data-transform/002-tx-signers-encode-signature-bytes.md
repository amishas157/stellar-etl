# H002: Transaction `tx_signers` exports signature bytes as fake account IDs

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_transactions.tx_signers` should contain the actual signing accounts or public keys for the transaction, or remain empty if the exporter cannot reliably recover them. Each element should correspond to a real signer identity, not to an encoded signature blob.

## Mechanism

`getTxSigners()` loops over `[]xdr.DecoratedSignature` but calls `strkey.Encode(strkey.VersionByteAccountID, sig.Signature)`. `sig.Signature` is the 64-byte signature payload, not the signer's 32-byte public key, so the function emits valid-looking `G...` strings derived from signature bytes and `TransformTransaction()` stores them in `tx_signers` for both classic and fee-bump envelopes.

## Trigger

Export any transaction with one or more signatures. Compare `tx_signers` to the transaction's real signer accounts: the emitted strings are computed from the raw signatures and do not identify the signing accounts.

## Target Code

- `internal/transform/transaction.go:getTxSigners:349-360` — encodes `sig.Signature` as an account ID
- `internal/transform/transaction.go:TransformTransaction:227-230` — fills `TxSigners` on the classic path
- `internal/transform/transaction.go:TransformTransaction:294-300` — fills `TxSigners` on the fee-bump path
- `internal/transform/schema.go:TransactionOutput:41-84` — labels the output field as `tx_signers`

## Evidence

The `xdr.DecoratedSignature` type contains only a 4-byte hint plus the signature bytes, but `getTxSigners()` never consults any signer key material. The helper directly encodes `sig.Signature` as an `AccountID`, and `transaction_test.go` hard-codes those synthetic `G...` strings as expected output, which means the mislabeled values currently ship as normal transaction data.

## Anti-Evidence

Recovering exact signer identities from decorated signatures alone may require more than the current helper has available, so the eventual fix could be a schema change rather than a code-only fix. Even so, the current exporter is still emitting wrong values under a field name that claims they are signers.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `getTxSigners()` at `transaction.go:349-360`. It receives `[]xdr.DecoratedSignature`, iterates over each, and calls `strkey.Encode(strkey.VersionByteAccountID, sig.Signature)`. The XDR `DecoratedSignature` struct (confirmed in `xdr_generated.go:25492`) contains `Hint SignatureHint` (4 bytes — last 4 bytes of the signer's public key) and `Signature Signature` (a `[]byte` alias holding the 64-byte ed25519 signature payload). The function encodes the 64-byte signature — not any key material — producing 108-character G... strings instead of standard 56-character Stellar addresses. These malformed strings are stored in `TransactionOutput.TxSigners` (schema.go:83) on both the classic path (line 270) and the fee-bump path (line 300).

### Code Paths Examined

- `internal/transform/transaction.go:getTxSigners:349-360` — Iterates `[]xdr.DecoratedSignature`, encodes `sig.Signature` (64-byte signature payload) via `strkey.Encode(VersionByteAccountID, ...)`. Never accesses `sig.Hint` or any public key source.
- `internal/transform/transaction.go:TransformTransaction:227-230` — Classic envelope path: calls `transaction.Envelope.Signatures()` and passes to `getTxSigners()`, stores result in `TxSigners`.
- `internal/transform/transaction.go:TransformTransaction:295-300` — Fee-bump path: calls `getTxSigners(transaction.Envelope.FeeBump.Signatures)`, overwrites `TxSigners` with fee-bump signer data (same encoding bug).
- `internal/transform/schema.go:83` — `TxSigners []string json:"tx_signers"` — the output field labeled as signer identities.
- `internal/transform/transaction_test.go:112,143,176,216` — Tests hardcode the 108-char encoded-signature strings as expected `TxSigners` values, confirming the bug is baked into the test suite.
- `stellar/go-stellar-sdk xdr/xdr_generated.go:25492-25496` — `DecoratedSignature{Hint SignatureHint, Signature Signature}` where `Signature` is `[]byte` (the ed25519 signature, not a public key).
- `stellar/go-stellar-sdk strkey/main.go:143-175` — `Encode()` accepts any `[]byte` up to `maxPayloadSize` with no length validation, so 64-byte signatures are silently encoded.

### Findings

The bug is confirmed: `tx_signers` contains strkey-encoded signature bytes, not signer identities. Key evidence:

1. **Wrong source data**: `sig.Signature` is the 64-byte ed25519 cryptographic signature. A Stellar account public key is 32 bytes. The function uses the wrong field.
2. **Abnormal output length**: The encoded strings are 108 characters (64-byte payload → base32), not the standard 56 characters of a Stellar G... address (32-byte payload). Any consumer validating address length would reject these.
3. **No signer recovery attempted**: The `Hint` field (last 4 bytes of the actual signer's public key) is never accessed. No attempt is made to resolve hints against known signers.
4. **Tests enshrine the bug**: `transaction_test.go` hardcodes the 108-char encoded-signature strings as expected output, so the bug passes all existing tests.
5. **Affects all transactions**: Both classic (line 227) and fee-bump (line 295) code paths use the same broken helper.

This produces incorrect but plausible-looking data in a field that downstream compliance and audit systems would use to identify transaction signers. The field name `tx_signers` strongly implies signer identities, but the values are derived from signature bytes and cannot be used to identify any account.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go`
- **Setup**: Use existing test transaction fixtures. The current test data already demonstrates the bug — the hardcoded `TxSigners` values are 108-char strings derived from signatures.
- **Steps**: (1) Decode the test transaction envelope to extract the `DecoratedSignature`. (2) Show that `sig.Signature` is 64 bytes. (3) Show that `strkey.Encode(VersionByteAccountID, sig.Signature)` produces the 108-char string in the test expectation. (4) Show that the transaction's actual source account public key is a different 56-char G... address.
- **Assertion**: Assert that the `TxSigners` output does NOT match any actual signer public key on the transaction, and that the encoded strings are 108 characters (not the 56-character standard for account addresses). This proves the field contains encoded signature bytes, not signer identities.
