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
