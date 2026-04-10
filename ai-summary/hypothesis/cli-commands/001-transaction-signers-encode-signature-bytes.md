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
