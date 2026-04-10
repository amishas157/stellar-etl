# 001: `tx_signers` encodes signature bytes as fake account IDs

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` exports `tx_signers` values that look like Stellar `G...` account IDs, but they are actually `AccountID`-encoded Ed25519 signature payloads. The field therefore contains wrong signer identity data for both classic and fee-bump transactions, and downstream systems cannot safely use it for signer attribution or joins against account tables.

## Root Cause

`getTxSigners()` iterates `[]xdr.DecoratedSignature` and passes each `sig.Signature` byte slice to `strkey.Encode(strkey.VersionByteAccountID, ...)`. `DecoratedSignature.Signature` is the 64-byte signature payload, not a 32-byte public key, so the helper serializes signature blobs as fake account IDs and `TransformTransaction()` stores them in `TransactionOutput.TxSigners`.

## Reproduction

During normal export, `TransformTransaction()` calls `getTxSigners()` on `transaction.Envelope.Signatures()` for classic transactions and on `transaction.Envelope.FeeBump.Signatures` for fee-bump transactions. The exported value exactly matches `strkey.Encode(strkey.VersionByteAccountID, sig.Signature)` and is 108 characters long, which demonstrates that the field contains encoded signature bytes rather than real 56-character Stellar account IDs.

## Affected Code

- `internal/transform/transaction.go:227-230` — classic transaction path populates `TxSigners`
- `internal/transform/transaction.go:295-300` — fee-bump transaction path overwrites `TxSigners` the same way
- `internal/transform/transaction.go:349-360` — `getTxSigners()` encodes signature payload bytes as `AccountID` strings
- `internal/transform/schema.go:40-84` — schema exposes the mislabeled output field as `tx_signers`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTxSignersEncodeSignatureBytesNotPublicKeys`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestTxSignersEncodeSignatureBytesNotPublicKeys -v`
  4. Observe that `tx_signers[0]` equals the account-version encoding of the raw 64-byte signature payload and not a real signer address.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/strkey"
)

func TestTxSignersEncodeSignatureBytesNotPublicKeys(t *testing.T) {
	transactions, ledgerHeaders, err := makeTransactionTestInput()
	if err != nil {
		t.Fatalf("makeTransactionTestInput returned error: %v", err)
	}

	if len(transactions) == 0 {
		t.Fatal("expected transaction fixtures")
	}

	output, err := TransformTransaction(transactions[0], ledgerHeaders[0])
	if err != nil {
		t.Fatalf("TransformTransaction returned error: %v", err)
	}

	signatures := transactions[0].Envelope.Signatures()
	if len(signatures) != 1 {
		t.Fatalf("expected one decorated signature, got %d", len(signatures))
	}

	if len(signatures[0].Signature) != 64 {
		t.Fatalf("expected a 64-byte signature payload, got %d bytes", len(signatures[0].Signature))
	}

	if len(output.TxSigners) != 1 {
		t.Fatalf("expected one tx signer, got %d", len(output.TxSigners))
	}

	directlyEncoded, err := strkey.Encode(strkey.VersionByteAccountID, signatures[0].Signature)
	if err != nil {
		t.Fatalf("strkey.Encode returned error: %v", err)
	}

	if output.TxSigners[0] != directlyEncoded {
		t.Fatalf("tx_signers[0] does not match the signature payload encoding:\n  got:  %s\n  want: %s", output.TxSigners[0], directlyEncoded)
	}

	if len(output.TxSigners[0]) != 108 {
		t.Fatalf("expected encoded signature payload to be 108 characters, got %d", len(output.TxSigners[0]))
	}

	if len(output.Account) != 56 {
		t.Fatalf("expected exported source account to be a 56-character Stellar address, got %d", len(output.Account))
	}

	if output.TxSigners[0] == output.Account {
		t.Fatal("tx_signers unexpectedly equals the transaction account")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `tx_signers` should contain real signer identities (or stay empty / use a differently named field if signer identities cannot be derived from the envelope data).
- **Actual**: `tx_signers` contains 108-character `G...` strings produced by encoding raw 64-byte signature payloads with the `AccountID` version byte.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs `TransformTransaction()` on a real fixture and independently reproduces the exported value from the underlying signature bytes.
2. Realistic preconditions: YES — every signed classic transaction and every signed fee-bump envelope executes this path during export.
3. Bug vs by-design: BUG — the schema field is named `tx_signers`, and the code uses the `AccountID` strkey version byte; exporting signature blobs under that label is silent identity corruption, not a documented opaque-signature format.
4. Final severity: High — the field is structurally wrong and will mislead downstream attribution, audit, and analytics systems, but it does not directly corrupt monetary values.
5. In scope: YES — this is production export logic producing silent wrong output.
6. Test correctness: CORRECT — the test uses production fixtures and code, avoids mocks, and verifies the output against an independent encoding of the exact bytes consumed by the buggy helper.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Stop encoding `DecoratedSignature.Signature` as an account ID. If exact signer identities cannot be recovered from the available XDR, leave `tx_signers` empty or replace it with a differently named field that explicitly stores signature material; otherwise resolve signer identities from actual signer keys and export those addresses instead.
