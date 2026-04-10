# H001: Transaction signer export encodes signature payloads as account IDs

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_transactions` should emit `tx_signers` as the actual signer account IDs for the transaction, or leave the field empty when signer identities cannot be derived from the available XDR. The exported values should be stable account identifiers, not per-signature payloads.

## Mechanism

`getTxSigners` iterates over `[]xdr.DecoratedSignature` and calls `strkey.Encode(strkey.VersionByteAccountID, sig.Signature)`. `sig.Signature` is the 64-byte signature blob, not the 32-byte ed25519 public key, so the command serializes each signature payload as if it were an account ID. The resulting `tx_signers` values vary with the signature bytes themselves and do not identify the real signing accounts, which makes downstream signer attribution silently wrong.

## Trigger

Run `export_transactions` on any ledger range containing signed transactions, including ordinary ed25519-signed transactions or fee-bump transactions. Inspect the `tx_signers` field in the JSON output: it is derived from the signature bytes for that transaction rather than from the signer public keys.

## Target Code

- `internal/transform/transaction.go:227-230` — base path collects signatures from the envelope
- `internal/transform/transaction.go:270-300` — fee-bump path overwrites `TxSigners` with the same helper
- `internal/transform/transaction.go:349-358` — `getTxSigners` encodes `sig.Signature` as `VersionByteAccountID`
- `internal/transform/schema.go:41-84` — exported transaction schema promises `tx_signers`
- `cmd/export_transactions.go:34-65` — command writes the transformed transaction rows directly to output

## Evidence

The helper never inspects signer hints, source accounts, or threshold signers; it only feeds raw `DecoratedSignature.Signature` bytes into `strkey.Encode`. The transaction tests already assert very long `TxSigners` strings produced by this helper (`internal/transform/transaction_test.go:112-143,176-216`), which is consistent with encoding signature payload bytes instead of normal Stellar account IDs.

## Anti-Evidence

The command does not currently have enough information to recover signer identities perfectly in all cases, especially for arbitrary multisig sets. That means the safest fix may be to stop emitting this field, emit signer hints, or clearly rename it, rather than trying to infer full signer accounts from incomplete data.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-transform/001-tx-signers-encode-signature-bytes.md.gh-published`
**Failed At**: reviewer

### Trace Summary

The hypothesis correctly identifies that `getTxSigners` at `internal/transform/transaction.go:349-358` encodes 64-byte `sig.Signature` payloads via `strkey.Encode(strkey.VersionByteAccountID, ...)`, producing 108-character fake account IDs instead of the standard 56-character Stellar account IDs. The bug is real and confirmed by tracing the code path. However, this exact finding has already been confirmed and published.

### Code Paths Examined

- `internal/transform/transaction.go:349-361` — `getTxSigners` encodes `sig.Signature` (64 bytes) as `VersionByteAccountID`, confirmed
- `internal/transform/transaction.go:227-230` — base transaction path calls `getTxSigners(transaction.Envelope.Signatures())`
- `internal/transform/transaction.go:295-300` — fee-bump path overwrites `TxSigners` with `getTxSigners(transaction.Envelope.FeeBump.Signatures)`
- `internal/transform/transaction_test.go:112,143,176,216` — tests assert 108-char strings consistent with signature-byte encoding

### Why It Failed

This is a duplicate of an already-confirmed and published finding at `ai-summary/success/data-transform/001-tx-signers-encode-signature-bytes.md.gh-published`. The same bug was also flagged as duplicate in `ai-summary/fail/data-transform/008-tx-signers-encode-signatures-as-accounts.md`, `009-tx-signers-encode-signatures-as-accounts.md`, and `010-tx-signers-encode-signatures-as-accounts.md`. The underlying bug lives in `internal/transform/transaction.go` (data-transform subsystem), not in `cmd/` (cli-commands subsystem), so this is a cross-subsystem duplicate of the same root cause.

### Lesson Learned

The `getTxSigners` bug resides in `internal/transform/`, making it a data-transform finding. Hypotheses scoped to the cli-commands subsystem that target transform-layer code should be checked against the data-transform success/fail directories for duplicates.
