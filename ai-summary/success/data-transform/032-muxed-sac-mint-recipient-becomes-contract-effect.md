# 032: Muxed SAC mint recipient becomes contract effect

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`effects()` misclassifies Stellar Asset Contract mint events when the recipient is a valid muxed `M...` account. Instead of exporting `account_credited` for the underlying canonical `G...` account and preserving the muxed form in `address_muxed`, it falls back to `contract_credited`, attributes the row to the operation source account, and stores the real recipient under `details.contract`.

This is a structural data-corruption bug in exported effects. The output looks plausible, but the credited party, effect type, and muxed-address placement are all wrong for a valid SAC mint.

## Root Cause

`addInvokeHostFunctionEffects()` decides whether a SAC mint recipient is an account by calling `strkey.IsValidEd25519PublicKey(mintEvent.To)`. That validator only accepts canonical `G...` addresses. Valid muxed `M...` recipients therefore miss the account branch and fall into the contract fallback, which calls `addMuxed(source, EffectContractCredited, details)` using the operation source account instead of the event recipient.

## Reproduction

Create an `InvokeHostFunction` transaction whose Soroban metadata contains a Stellar Asset Contract `mint` event with a muxed `M...` recipient. Run the transaction through `transactionOperationWrapper.effects()`.

The exporter emits a single `contract_credited` effect on the admin/source account, leaves `address_muxed` empty, and writes the real `M...` recipient into `details["contract"]`. The credited recipient is therefore exported incorrectly even though the event itself is valid.

## Affected Code

- `internal/transform/effects.go:191-197` — `addMuxed()` correctly preserves canonical and muxed account forms when it is given the actual recipient
- `internal/transform/effects.go:1380-1394` — the mint branch only accepts `G...` via `IsValidEd25519PublicKey` and otherwise falls back to `EffectContractCredited` on the source account

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestMuxedSACMintRecipientBecomesContractEffect`
- **Test language**: go
- **How to run**: Create the target test file with the test body below, then run `go build ./...` and `go test ./internal/transform/... -run TestMuxedSACMintRecipientBecomesContractEffect -v`.

### Test Body

```go
package transform

import (
	"math/big"
	"testing"

	"github.com/stellar/go-stellar-sdk/strkey"
	"github.com/stellar/go-stellar-sdk/support/contractevents"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestMuxedSACMintRecipientBecomesContractEffect(t *testing.T) {
	recipient := "GDQNY3PBOJOKYZSRMK2S7LHHGWZIUISD4QORETLMXEWXBI7KFZZMKTL3"
	admin := "GDRW375MAYR46ODGF2WGANQC2RRZL7O246DYHHCGWTV2RE7IHE2QUQLD"

	muxedRecipient, err := xdr.MuxedAccountFromAccountId(recipient, 12345)
	if err != nil {
		t.Fatalf("build muxed recipient: %v", err)
	}
	muxedAddress, err := muxedRecipient.GetAddress()
	if err != nil {
		t.Fatalf("encode muxed recipient: %v", err)
	}

	if strkey.IsValidEd25519PublicKey(muxedAddress) {
		t.Fatalf("expected %s to be rejected as a plain G-address", muxedAddress)
	}
	if !strkey.IsValidMuxedAccountEd25519PublicKey(muxedAddress) {
		t.Fatalf("expected %s to be a valid muxed account", muxedAddress)
	}

	tx := makeInvocationTransaction(
		"",
		muxedAddress,
		admin,
		xdr.MustNewCreditAsset("TESTER", admin),
		big.NewInt(12345),
		contractevents.EventTypeMint,
	)

	operation := transactionOperationWrapper{
		index:          0,
		transaction:    tx,
		operation:      tx.Envelope.Operations()[0],
		ledgerSequence: 1,
		network:        networkPassphrase,
	}

	effects, err := operation.effects()
	if err != nil {
		t.Fatalf("effects(): %v", err)
	}
	if len(effects) != 1 {
		t.Fatalf("expected 1 effect, got %d", len(effects))
	}

	effect := effects[0]
	if effect.Type != int32(EffectAccountCredited) ||
		effect.Address != recipient ||
		!effect.AddressMuxed.Valid ||
		effect.AddressMuxed.String != muxedAddress ||
		effect.Details["contract"] != nil {
		t.Fatalf("expected account_credited for %s with address_muxed=%s and no contract detail, got %+v", recipient, muxedAddress, effect)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: a SAC mint to a muxed recipient should export `account_credited`, set `address` to the canonical `G...` account, preserve the original `M...` value in `address_muxed`, and avoid any contract fallback metadata.
- **Actual**: the exporter writes `contract_credited`, sets `address` to the admin/source account, leaves `address_muxed` null, and stores the real muxed recipient in `details["contract"]`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs the real `transactionOperationWrapper.effects()` path and reaches the mint branch in `addInvokeHostFunctionEffects()`.
2. Realistic preconditions: YES — `SC_ADDRESS_TYPE_MUXED_ACCOUNT` is a supported XDR address type, and the upstream SAC event tooling round-trips it to `M...` strings.
3. Bug vs by-design: BUG — the code already has `addMuxed()` for account-shaped outputs, so treating a muxed account recipient as a contract is inconsistent with the surrounding effect model.
4. Final severity: High — the bug corrupts exported effect identity and classification but does not directly alter monetary values.
5. In scope: YES — this is a concrete data-correctness defect in `internal/transform/`.
6. Test correctness: CORRECT — the test constructs a valid muxed recipient, uses production event generation and parsing, and fails on the wrong exported fields.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Treat valid muxed account recipients as account recipients in SAC effect generation. The mint, transfer, and clawback account checks should accept both `G...` and `M...` addresses, then route muxed recipients through `addMuxed()` using the event participant rather than the operation source account.
