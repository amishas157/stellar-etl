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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTxSignersEncodeSignaturesNotPublicKeys"
**Test Language**: Go

### Demonstration

The test calls `getTxSigners()` with a `DecoratedSignature` containing a 64-byte Ed25519 signature and verifies the returned string is 108 characters long (not the standard 56-character account ID length). It also confirms the returned value does not match the transaction's actual source account address. This proves `tx_signers` contains encoded signature material rather than signer identities, producing fake `G...` addresses that look valid but correspond to no real Stellar account.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestTxSignersEncodeSignaturesNotPublicKeys demonstrates that getTxSigners()
// encodes the raw 64-byte Ed25519 signature as a G... account ID instead of
// encoding the 32-byte signer public key. This produces a 108-character string
// (64 bytes encoded) rather than the standard 56-character account ID (32 bytes).
func TestTxSignersEncodeSignaturesNotPublicKeys(t *testing.T) {
	// Use the same signature bytes from the existing test fixtures
	sig := xdr.DecoratedSignature{
		Hint: xdr.SignatureHint{99, 66, 175, 143},
		Signature: xdr.Signature{
			244, 107, 139, 92, 189, 156, 207, 79, 84, 56, 2, 70, 75, 22, 237,
			50, 100, 242, 159, 177, 27, 240, 66, 122, 182, 45, 189, 78, 5, 127,
			26, 61, 179, 238, 229, 76, 32, 206, 122, 13, 154, 133, 148, 149, 29,
			250, 48, 132, 44, 86, 163, 56, 32, 44, 75, 87, 226, 251, 76, 4, 59,
			182, 132, 8,
		},
	}

	signers, err := getTxSigners([]xdr.DecoratedSignature{sig})
	if err != nil {
		t.Fatalf("getTxSigners returned unexpected error: %v", err)
	}

	if len(signers) != 1 {
		t.Fatalf("expected 1 signer, got %d", len(signers))
	}

	signer := signers[0]

	// A valid Stellar account ID (G...) is always 56 characters because it
	// encodes exactly 32 bytes (Ed25519 public key). The signature is 64 bytes,
	// so encoding it as an account ID produces a 108-character string.
	const validAccountIDLen = 56

	if len(signer) == validAccountIDLen {
		t.Fatalf("signer string is 56 chars — expected longer (108) because "+
			"getTxSigners encodes the 64-byte signature, not a 32-byte public key. Got: %s", signer)
	}

	// The returned string is 108 characters — proof it encodes 64 bytes of
	// signature material, not the 32-byte signer public key.
	const expectedBuggyLen = 108
	if len(signer) != expectedBuggyLen {
		t.Errorf("expected signer string length %d (64-byte signature encoded as account ID), got %d: %s",
			expectedBuggyLen, len(signer), signer)
	}

	// Additionally verify the source account's real address is NOT in the output.
	// The transaction's source account (testAccount1) has a known 56-char address.
	if signer == testAccount1Address {
		t.Errorf("signer unexpectedly matches the source account address %s — "+
			"expected encoded signature material instead", testAccount1Address)
	}

	t.Logf("BUG CONFIRMED: getTxSigners() returns %d-char string (encoded 64-byte signature) "+
		"instead of %d-char account ID (32-byte public key)", len(signer), validAccountIDLen)
	t.Logf("Returned signer: %s", signer)
	t.Logf("Source account:  %s", testAccount1Address)
}
```

### Test Output

```
=== RUN   TestTxSignersEncodeSignaturesNotPublicKeys
    data_integrity_poc_test.go:62: BUG CONFIRMED: getTxSigners() returns 108-char string (encoded 64-byte signature) instead of 56-char account ID (32-byte public key)
    data_integrity_poc_test.go:64: Returned signer: GD2GXC24XWOM6T2UHABEMSYW5UZGJ4U7WEN7AQT2WYW32TQFP4ND3M7O4VGCBTT2BWNILFEVDX5DBBBMK2RTQIBMJNL6F62MAQ53NBAIXUDA
    data_integrity_poc_test.go:65: Source account:  GCEODJVUUVYVFD5KT4TOEDTMXQ76OPFOQC2EMYYMLPXQCUVPOB6XRWPQ
--- PASS: TestTxSignersEncodeSignaturesNotPublicKeys (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.906s
```
