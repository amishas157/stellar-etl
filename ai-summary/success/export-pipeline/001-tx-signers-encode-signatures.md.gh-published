# 001: tx_signers encodes signature bytes as fake account IDs

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` exports `tx_signers` by base32-encoding each `DecoratedSignature.Signature` with the Stellar account-ID version byte. The result is a plausible-looking `G...` string that is not a signer identity, so every signed transaction can carry structurally wrong signer data in the export.

## Root Cause

`getTxSigners()` treats the 64-byte Ed25519 signature payload as if it were a 32-byte account public key and passes it to `strkey.Encode(strkey.VersionByteAccountID, ...)`. Both the classic and fee-bump transaction paths copy that output directly into `TransactionOutput.TxSigners`.

## Reproduction

Export any signed transaction or fee-bump transaction. The exported `tx_signers` field contains 108-character `G...` values derived from signature blobs, not the actual signer account IDs/public keys, so downstream joins and signer attribution silently operate on bogus identities.

## Affected Code

- `internal/transform/transaction.go:227-230` — classic transaction path populates `TxSigners` from `getTxSigners(transaction.Envelope.Signatures())`
- `internal/transform/transaction.go:294-300` — fee-bump path overwrites `TxSigners` from `transaction.Envelope.FeeBump.Signatures`
- `internal/transform/transaction.go:349-360` — `getTxSigners()` encodes `sig.Signature` as `VersionByteAccountID`
- `internal/transform/schema.go:83` — exported field is explicitly named `tx_signers`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTxSignersEncodeSignaturesNotPublicKeys`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

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

	const validAccountIDLen = 56
	if len(signer) == validAccountIDLen {
		t.Fatalf("signer string is 56 chars; expected 108 because getTxSigners encodes the 64-byte signature, not a 32-byte public key: %s", signer)
	}

	const expectedBuggyLen = 108
	if len(signer) != expectedBuggyLen {
		t.Errorf("expected signer string length %d (64-byte signature encoded as account ID), got %d: %s",
			expectedBuggyLen, len(signer), signer)
	}

	if signer == testAccount1Address {
		t.Errorf("signer unexpectedly matches the source account address %s; expected encoded signature material instead", testAccount1Address)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `tx_signers` should either contain real signer identities or stay empty when the exporter cannot derive them from `DecoratedSignature`.
- **Actual**: `tx_signers` contains synthetic `G...` strings derived from cryptographic signature bytes, not signer identities.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC invokes the exact production helper that both transaction export branches use to populate `TransactionOutput.TxSigners`.
2. Realistic preconditions: YES — every normally signed Stellar transaction includes `DecoratedSignature` values, so the input state is routine.
3. Bug vs by-design: BUG — the field name, JSON tag, and account-ID encoding all present the values as signer identities, but the code never reads any signer public key.
4. Final severity: High — this is structural export corruption in a signer-identity field, not a monetary miscalculation.
5. In scope: YES — the export pipeline emits wrong but plausible data without failing.
6. Test correctness: CORRECT — the assertion checks the exported helper output shape against known account-ID semantics and rules out the transaction source account as the returned value.
7. Alternative explanations: NONE — the only bytes encoded are `sig.Signature`, and that field is cryptographic signature material rather than a public key.
8. Novelty: PREVIOUSLY REPORTED IN REPO HISTORY — the issue is still real; duplicate handling is external to this review step.

## Suggested Fix

Do not encode `DecoratedSignature.Signature` with `VersionByteAccountID`. Either remove `tx_signers`, leave it empty unless a real signer key can be resolved, or rename/retype the field to explicitly store raw signature material in a non-account format.
