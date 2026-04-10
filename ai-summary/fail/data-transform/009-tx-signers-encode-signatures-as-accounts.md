# H004: `tx_signers` encodes signatures as fake account IDs

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`tx_signers` should contain signer identities that downstream systems can use as Stellar account IDs, or it should remain empty if the transform cannot reliably derive those identities from the transaction. The exported strings should not look like account IDs while actually encoding unrelated bytes.

## Mechanism

`getTxSigners()` iterates over `[]xdr.DecoratedSignature` and feeds `sig.Signature` directly into `strkey.Encode(...VersionByteAccountID...)`. That value is the raw Ed25519 signature blob, not a public key, so the code emits long `G...` strings that look like account IDs but are just base32-encoded signatures. The fee-bump branch reuses the same helper, so both normal and fee-bump transaction exports carry the same corrupted identity field.

## Trigger

Export any signed transaction. The resulting `tx_signers` array will contain 108-character pseudo-account strings derived from signature bytes instead of 56-character Stellar account IDs for the actual signers.

## Target Code

- `internal/transform/schema.go:83` — schema advertises the field as `tx_signers`
- `internal/transform/transaction.go:227-270` — normal transaction path populates `TxSigners` from `getTxSigners`
- `internal/transform/transaction.go:295-300` — fee-bump path reuses the same helper
- `internal/transform/transaction.go:336-346` — nearby `formatSigners()` shows the correct pattern: `SignerKey.Address()`
- `internal/transform/transaction.go:349-357` — helper encodes `sig.Signature` as an account ID
- `internal/transform/transaction_test.go:176` — existing fixture already locks in one of the bogus 108-character outputs

## Evidence

The helper never inspects a signer key; it only sees `xdr.DecoratedSignature` and passes `sig.Signature` to `strkey.Encode` with the account-ID version byte. In contrast, `formatSigners()` correctly converts `xdr.SignerKey` values with `key.Address()`. The test fixture at `transaction_test.go:176` expects a `tx_signers` string that is 108 characters long, which is incompatible with a real Stellar account ID and strongly indicates the field is encoding the signature blob itself.

## Anti-Evidence

If the original intent of `tx_signers` was "opaque signature material encoded as a string," the current implementation is consistent with that narrower goal. But the schema name, account-ID version byte, and `G...` output format all present the values as signer identities, which makes the exported data materially misleading.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-transform/001-tx-signers-encode-signature-bytes.md.gh-published`
**Failed At**: reviewer

### Trace Summary

This hypothesis is an exact duplicate of the already-confirmed and published finding in `ai-summary/success/data-transform/001-tx-signers-encode-signature-bytes.md.gh-published`. That success file documents the identical bug: `getTxSigners()` encodes `sig.Signature` (64-byte Ed25519 signature payload) using `strkey.Encode(strkey.VersionByteAccountID, ...)`, producing 108-character fake `G...` account IDs instead of real 56-character signer public keys. The finding has already been through the full pipeline including PoC verification and adversarial review. Additionally, this same hypothesis was previously rejected as duplicate in `ai-summary/fail/data-transform/008-tx-signers-encode-signatures-as-accounts.md`.

### Code Paths Examined

- `internal/transform/transaction.go:349-357` — `getTxSigners()` helper, same code path identified in success/001 and fail/008

### Why It Failed

This is a duplicate of an already-confirmed, PoC-verified, and published finding. The success file `001-tx-signers-encode-signature-bytes.md.gh-published` covers the identical mechanism, identical affected code paths, and identical impact. This hypothesis was also previously reviewed and rejected as a duplicate in fail/008.

### Lesson Learned

This hypothesis was generated despite the identical bug already existing in `ai-summary/success/`. The hypothesis generator should check success files before proposing new hypotheses targeting the same code paths.
