# 034: Muxed SAC burn sender becomes contract effect

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`transactionOperationWrapper.effects()` misclassifies Stellar Asset Contract burn events when the debited holder is a valid muxed `M...` account. Instead of exporting `account_debited` for the underlying canonical `G...` account and preserving the muxed form in `address_muxed`, it falls back to `contract_debited`, attributes the row to the operation source account, and stores the real holder under `details.contract`.

This is structural data corruption in exported effects. The row still looks plausible, but the effect type, affected account, and muxed-address placement are all wrong for a valid SAC burn event.

## Root Cause

`addInvokeHostFunctionEffects()` decides whether a SAC burn sender is an account by calling `strkey.IsValidEd25519PublicKey(burnEvent.From)`. That validator only accepts canonical `G...` addresses. Valid muxed `M...` holders therefore miss the account branch and fall into the contract fallback, which calls `addMuxed(source, EffectContractDebited, details)` using the operation source account instead of the burned holder.

## Reproduction

Create an `InvokeHostFunction` transaction whose Soroban metadata contains a Stellar Asset Contract `burn` event with a muxed `M...` holder in the `from` position. Run the transaction through `transactionOperationWrapper.effects()`.

The exporter emits a single `contract_debited` effect on the admin/source account, leaves `address_muxed` empty, and writes the real muxed holder into `details["contract"]`. The debited account is therefore exported incorrectly even though the event itself is valid.

## Affected Code

- `internal/transform/effects.go:191-197` — `addMuxed()` already preserves canonical and muxed account forms when given the real participant
- `internal/transform/effects.go:1414-1428` — the burn branch only accepts `G...` via `IsValidEd25519PublicKey` and otherwise falls back to `EffectContractDebited` on the source account

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestMuxedSACBurnSenderBecomesContractEffect`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go build ./...` and `go test ./internal/transform/... -run TestMuxedSACBurnSenderBecomesContractEffect -v`.

### Test Body

```go
package transform

import (
	"math/big"
	"testing"

	"github.com/stellar/go-stellar-sdk/keypair"
	"github.com/stellar/go-stellar-sdk/strkey"
	"github.com/stellar/go-stellar-sdk/support/contractevents"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestMuxedSACBurnSenderBecomesContractEffect(t *testing.T) {
	holder := keypair.MustRandom().Address()

	muxedHolder := &strkey.MuxedAccount{}
	if err := muxedHolder.SetAccountID(holder); err != nil {
		t.Fatalf("set account id: %v", err)
	}
	muxedHolder.SetID(42)

	muxedAddress, err := muxedHolder.Address()
	if err != nil {
		t.Fatalf("get muxed address: %v", err)
	}
	if !strkey.IsValidMuxedAccountEd25519PublicKey(muxedAddress) {
		t.Fatalf("expected valid muxed address, got %q", muxedAddress)
	}
	if strkey.IsValidEd25519PublicKey(muxedAddress) {
		t.Fatalf("expected muxed address %q to fail G-address validation", muxedAddress)
	}

	admin := keypair.MustRandom().Address()
	asset := xdr.MustNewCreditAsset("TESTER", admin)

	tx := makeInvocationTransaction(
		muxedAddress,
		"",
		admin,
		asset,
		big.NewInt(12345),
		contractevents.EventTypeBurn,
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
		t.Fatalf("generate effects: %v", err)
	}
	if len(effects) != 1 {
		t.Fatalf("expected 1 effect, got %d", len(effects))
	}

	effect := effects[0]
	if effect.Type != int32(EffectContractDebited) {
		t.Fatalf("expected buggy contract_debited effect, got %s (%d)", effect.TypeString, effect.Type)
	}
	if effect.Address != admin {
		t.Fatalf("expected buggy address %s, got %s", admin, effect.Address)
	}
	if effect.Address == holder {
		t.Fatalf("expected canonical holder %s to be lost, but it was preserved", holder)
	}
	if effect.AddressMuxed.Valid {
		t.Fatalf("expected address_muxed to be empty, got %s", effect.AddressMuxed.String)
	}
	if got := effect.Details["contract"]; got != muxedAddress {
		t.Fatalf("expected muxed holder to be misfiled in details.contract, got %#v", got)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: a SAC burn from a muxed holder should export `account_debited`, set `address` to the canonical `G...` account, preserve the original `M...` value in `address_muxed`, and avoid any contract fallback metadata.
- **Actual**: the exporter writes `contract_debited`, sets `address` to the admin/source account, leaves `address_muxed` null, and stores the real muxed holder in `details["contract"]`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs the real `transactionOperationWrapper.effects()` path and reaches the burn branch in `addInvokeHostFunctionEffects()`.
2. Realistic preconditions: YES — muxed accounts are a supported `SCAddress` type in upstream XDR and the SDK converts them to `M...` strings during SAC event parsing.
3. Bug vs by-design: BUG — the effects schema already has `address_muxed`, and `addMuxed()` exists specifically to preserve muxed account identity, so treating a valid muxed account as a contract is inconsistent with the surrounding model.
4. Final severity: High — the bug corrupts exported effect identity and classification but does not directly alter monetary values.
5. In scope: YES — this is a concrete data-correctness defect in `internal/transform/`.
6. Test correctness: CORRECT — the test uses the in-repo event generator and production effect path, and it checks the emitted row fields directly.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Treat valid muxed account holders as account participants in SAC effect generation. The transfer, mint, clawback, and burn account checks should accept both `G...` and `M...` addresses, then decode muxed participants and route them through `addMuxed()` using the event participant rather than the operation source account.
