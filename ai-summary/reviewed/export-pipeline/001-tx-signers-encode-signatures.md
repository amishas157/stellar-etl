# H001: `tx_signers` encodes signature blobs as fake account IDs

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`tx_signers` should contain the actual signer account IDs/public keys that authorized the transaction, or remain empty if the exporter cannot derive them from the available XDR. It should never emit a plausible-looking account string that is actually derived from signature bytes instead of a signer identity.

## Mechanism

`TransformTransaction()` calls `getTxSigners()`, and `getTxSigners()` feeds `xdr.DecoratedSignature.Signature` directly into `strkey.Encode(...VersionByteAccountID...)`. `DecoratedSignature.Signature` is the raw Ed25519 signature payload, not the public key, so the exporter serializes synthetic `G...` strings that do not identify any real signer while still looking valid to downstream analytics.

## Trigger

Export any normal or fee-bump transaction that contains at least one decorated signature.

## Target Code

- `internal/transform/transaction.go:227-230` — transaction export always populates `TxSigners`
- `internal/transform/transaction.go:294-300` — fee-bump path overwrites `TxSigners` the same way
- `internal/transform/transaction.go:349-360` — `getTxSigners()` encodes `sig.Signature` as an account ID

## Evidence

The implementation never reads a signer public key; it only sees `Hint` and `Signature` from `xdr.DecoratedSignature`, then encodes the signature blob as if it were a 32-byte account payload. Repository tests already expect very long synthetic `G...` strings in `TxSigners`, which shows this code path is exercised rather than dead.

## Anti-Evidence

`ExtraSigners` uses `xdr.SignerKey.Address()` correctly, so not every signer-related field is broken. A reviewer could argue `tx_signers` was intended to store encoded signatures, but the field name, JSON tag, and account-ID version byte all indicate signer identities, not signature material.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformTransaction()` at line 227 calls `getTxSigners(transaction.Envelope.Signatures())` which passes the `DecoratedSignature` slice to `getTxSigners()`. That function (lines 349-361) iterates each signature and calls `strkey.Encode(strkey.VersionByteAccountID, sig.Signature)`, encoding the raw 64-byte Ed25519 signature with the account-ID version byte. This produces a 108-character `G...` string (versus 56 characters for a real account ID). The fee-bump path at line 295 does the same for `FeeBump.Signatures`. The `formatSigners()` function at line 336 shows the correct pattern — it calls `key.Address()` on `xdr.SignerKey` values — confirming the codebase knows how to extract real signer identities.

### Code Paths Examined

- `internal/transform/transaction.go:227-230` — calls `getTxSigners()` with `transaction.Envelope.Signatures()` and assigns result to `txSigners`
- `internal/transform/transaction.go:270` — assigns `txSigners` to `TransactionOutput.TxSigners`
- `internal/transform/transaction.go:294-300` — fee-bump path: calls `getTxSigners()` with `transaction.Envelope.FeeBump.Signatures` and overwrites `TxSigners`
- `internal/transform/transaction.go:349-361` — `getTxSigners()` encodes `sig.Signature` (64-byte Ed25519 signature) as a `G...` account ID using `strkey.Encode(strkey.VersionByteAccountID, ...)`
- `internal/transform/transaction.go:336-347` — `formatSigners()` correctly uses `key.Address()` on `xdr.SignerKey` values for `ExtraSigners`, showing the intended pattern
- `internal/transform/schema.go:83` — `TxSigners []string \`json:"tx_signers"\`` field name and JSON tag both indicate signer identities
- `internal/transform/transaction_test.go:112,143,176,216` — tests expect 108-character `G...` strings, confirming the bug is exercised and baked into expectations

### Findings

The bug is confirmed. `xdr.DecoratedSignature` contains two fields: `Hint` (4-byte signer key hint) and `Signature` (64-byte Ed25519 cryptographic signature). The `getTxSigners()` function takes the `Signature` field — which is the cryptographic signature, not a public key — and encodes it using `strkey.Encode(strkey.VersionByteAccountID, ...)`. This produces a `G...`-prefixed string that:

1. **Is the wrong length**: 108 characters instead of the standard 56 characters for a Stellar account ID (because 64 bytes are encoded instead of 32).
2. **Contains no signer identity information**: The encoded value is the signature payload, not the signer's public key.
3. **Uses misleading encoding**: `VersionByteAccountID` makes the output look like a valid Stellar account address, but it doesn't correspond to any real account.

Every exported transaction populates `tx_signers` with these fake account IDs. Downstream consumers (BigQuery analytics, compliance reporting) querying `tx_signers` to identify transaction signers will get meaningless values. Any JOIN against an accounts table would silently produce zero matches.

The correct behavior from `DecoratedSignature` is limited — you cannot recover the full 32-byte public key from a 4-byte hint alone. The field should either store the raw signature as hex/base64 with a clearly non-account-ID format, or be omitted entirely, or require a separate signer resolution step.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go`
- **Setup**: Use the existing test transaction fixtures that contain `DecoratedSignature` entries (lines 285-290). The test data already has a 64-byte `Signature` field.
- **Steps**: Call `getTxSigners()` with the test `DecoratedSignature` slice. Measure the length of the returned strings.
- **Assertion**: Assert that each returned string is 108 characters (proving it encodes 64 bytes, not 32). Assert that the returned string does NOT match any known signer public key from the test transaction's source account. This demonstrates the field contains signature material, not signer identities. Compare against the correct 56-character account ID that `formatSigners()` would produce for the same transaction's actual signers.
