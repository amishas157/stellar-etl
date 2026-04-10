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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTxSignersEncodeSignatureBytesNotPublicKeys"
**Test Language**: Go

### Demonstration

The test calls the production `getTxSigners()` function with the same `DecoratedSignature` fixture used in the existing test suite. It proves that the 64-byte signature payload is encoded as a 108-character G... string (instead of the standard 56-character Stellar address from a 32-byte public key), that this output exactly matches independently encoding `sig.Signature` via `strkey.Encode(VersionByteAccountID, ...)`, and that it does NOT match the transaction's actual source account (`GCEODJVUUVYVFD5KT4TOEDTMXQ76OPFOQC2EMYYMLPXQCUVPOB6XRWPQ`).

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/strkey"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestTxSignersEncodeSignatureBytesNotPublicKeys demonstrates that getTxSigners()
// encodes the 64-byte ed25519 signature payload as a G... account ID, rather than
// encoding the actual signer's 32-byte public key. This produces 108-character strings
// that look like Stellar addresses but actually contain signature data.
func TestTxSignersEncodeSignatureBytesNotPublicKeys(t *testing.T) {
	// The signature from the test fixtures (64 bytes — an ed25519 signature, NOT a public key)
	sig := xdr.DecoratedSignature{
		Hint: xdr.SignatureHint{99, 66, 175, 143},
		Signature: xdr.Signature{
			244, 107, 139, 92, 189, 156, 207, 79, 84, 56, 2, 70, 75, 22, 237, 50,
			100, 242, 159, 177, 27, 240, 66, 122, 182, 45, 189, 78, 5, 127, 26, 61,
			179, 238, 229, 76, 32, 206, 122, 13, 154, 133, 148, 149, 29, 250, 48, 132,
			44, 86, 163, 56, 32, 44, 75, 87, 226, 251, 76, 4, 59, 182, 132, 8,
		},
	}

	// 1. Verify the signature payload is 64 bytes (not 32 bytes like a public key)
	if len(sig.Signature) != 64 {
		t.Fatalf("expected signature to be 64 bytes, got %d", len(sig.Signature))
	}

	// 2. Call the production function
	signers, err := getTxSigners([]xdr.DecoratedSignature{sig})
	if err != nil {
		t.Fatalf("getTxSigners returned error: %v", err)
	}

	// 3. The output string is 108 characters — NOT the 56 characters of a real Stellar G... address
	const standardStellarAddressLen = 56
	if len(signers[0]) == standardStellarAddressLen {
		t.Fatalf("expected signer string to NOT be %d chars (standard address length), but it was", standardStellarAddressLen)
	}
	if len(signers[0]) != 108 {
		t.Fatalf("expected signer string to be 108 chars (encoded 64-byte signature), got %d", len(signers[0]))
	}
	t.Logf("signer string length: %d (standard is %d)", len(signers[0]), standardStellarAddressLen)

	// 4. Independently reproduce: encoding the 64-byte signature as an AccountID yields the same string
	directlyEncoded, err := strkey.Encode(strkey.VersionByteAccountID, sig.Signature)
	if err != nil {
		t.Fatalf("direct strkey.Encode failed: %v", err)
	}
	if signers[0] != directlyEncoded {
		t.Errorf("getTxSigners output doesn't match direct encoding of signature bytes:\n  got:  %s\n  want: %s", signers[0], directlyEncoded)
	}

	// 5. Show that the output does NOT match the transaction's actual source account
	// testAccount1Address is the source account used in the test fixtures
	actualSourceAccount := testAccount1Address // "GCEODJVUUVYVFD5KT4TOEDTMXQ76OPFOQC2EMYYMLPXQCUVPOB6XRWPQ"
	if signers[0] == actualSourceAccount {
		t.Fatal("signer string unexpectedly equals the actual source account — hypothesis would be disproved")
	}
	t.Logf("source account:  %s (56 chars)", actualSourceAccount)
	t.Logf("tx_signers[0]:   %s (108 chars)", signers[0])
	t.Log("CONFIRMED: tx_signers contains strkey-encoded signature bytes, not signer identities")
}
```

### Test Output

```
=== RUN   TestTxSignersEncodeSignatureBytesNotPublicKeys
    data_integrity_poc_test.go:45: signer string length: 108 (standard is 56)
    data_integrity_poc_test.go:62: source account:  GCEODJVUUVYVFD5KT4TOEDTMXQ76OPFOQC2EMYYMLPXQCUVPOB6XRWPQ (56 chars)
    data_integrity_poc_test.go:63: tx_signers[0]:   GD2GXC24XWOM6T2UHABEMSYW5UZGJ4U7WEN7AQT2WYW32TQFP4ND3M7O4VGCBTT2BWNILFEVDX5DBBBMK2RTQIBMJNL6F62MAQ53NBAIXUDA (108 chars)
    data_integrity_poc_test.go:64: CONFIRMED: tx_signers contains strkey-encoded signature bytes, not signer identities
--- PASS: TestTxSignersEncodeSignatureBytesNotPublicKeys (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.780s
```
