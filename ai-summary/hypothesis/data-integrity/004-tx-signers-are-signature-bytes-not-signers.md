# H004: `tx_signers` exports raw signature blobs as account IDs

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`tx_signers` should contain actual signer identifiers for the transaction, or be omitted if the transform cannot derive them reliably. It should never emit account-shaped strings derived from the signature payload itself, because those values do not identify any Stellar signer and cannot be joined against account tables.

## Mechanism

`getTxSigners()` iterates `xdr.DecoratedSignature` values and calls `strkey.Encode(strkey.VersionByteAccountID, sig.Signature)`. But `DecoratedSignature.Signature` is the actual signature bytes, while the public-key hint is only the trailing 4-byte `Hint`; `strkey.Encode` accepts arbitrary payload sizes up to its max and therefore happily turns 64-byte signatures into long `G...` strings that look plausible but are not valid 32-byte account IDs. The JSON export thus silently publishes signer-looking garbage for every signed transaction.

## Trigger

Run `export_transactions` on any normal or fee-bump transaction with at least one signature and inspect the `tx_signers` array.

## Target Code

- `internal/transform/transaction.go:227-230` — normal transaction path populates `TxSigners`
- `internal/transform/transaction.go:295-300` — fee-bump path overwrites `TxSigners` from fee-bump signatures
- `internal/transform/transaction.go:349-357` — encodes `sig.Signature` as `VersionByteAccountID`
- `internal/transform/transaction_test.go:112,143,176,216` — tests currently lock in long `G...` outputs for `tx_signers`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/xdr_generated.go:25489-25495` — `DecoratedSignature` contains a 4-byte hint plus the actual signature bytes
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/strkey/main.go:148-153` — `strkey.Encode` accepts any payload up to the library max, not just 32-byte account keys

## Evidence

The checked-in tests already show `tx_signers` values far longer than a canonical 56-character ed25519 account strkey, which is exactly what you would expect after encoding a 64-byte signature instead of a 32-byte public key. The field name and JSON tag both promise signer identity, but the implementation only has signature material and converts it into account-shaped noise.

## Anti-Evidence

This code path mirrors logic found in the upstream Go ingest/processor packages, so there may be historical compatibility pressure to keep the current output format. If downstream consumers already treat `tx_signers` as opaque signature-derived strings instead of real signer IDs, the practical blast radius is smaller than the field name suggests.
