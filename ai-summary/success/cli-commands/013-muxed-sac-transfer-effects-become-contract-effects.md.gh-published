# 013: Muxed SAC transfer effects become contract effects

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`export_effects` routes invoke-host-function events through `transactionOperationWrapper.effects()`, which in turn calls `addInvokeHostFunctionEffects()` for Stellar Asset Contract balance changes. When a SAC `transfer` endpoint is a valid muxed account, the code rejects its `M...` address as a non-account, emits `contract_debited` instead of `account_debited`, attributes the row to the operation source account, and stores the muxed account in `details.contract`.

This is real structural corruption, not a test artifact. The SDK path that parses SAC topics explicitly preserves muxed endpoints as `M...` strings, and the effects layer already has `addMuxed()` helpers for preserving canonical `G...` plus `address_muxed` elsewhere.

## Root Cause

`addInvokeHostFunctionEffects()` decides whether SAC participants are accounts with `strkey.IsValidEd25519PublicKey(...)`. That helper only accepts canonical `G...` account IDs, so valid muxed `M...` account addresses fall into the fallback branch intended for contracts. That fallback calls `addMuxed(source, EffectContractDebited, details)` / `addMuxed(source, EffectContractCredited, ...)`, rebinding the effect to the operation source account and populating `details["contract"]` with the muxed account string.

## Reproduction

Create an invoke-host-function transaction whose SAC `transfer` event has a muxed `from` participant and a normal account `to` participant. Run the production `operation.effects()` path used by `TransformEffect()`: the first exported effect row is emitted as `contract_debited`, its top-level `address` is the operation source account instead of the underlying `G...` account, `address_muxed` is empty, and `details.contract` contains the muxed `M...` address.

## Affected Code

- `internal/transform/effects.go:addMuxed:191-197` — canonicalizes muxed accounts into `address` + `address_muxed`, which the SAC path fails to use for muxed participants
- `internal/transform/effects.go:addInvokeHostFunctionEffects:1322-1376` — classifies SAC transfer endpoints with `IsValidEd25519PublicKey` and sends muxed accounts down the contract-effect branch
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/utils.go:parseBalanceChangeEvent:15-51` — parses SAC topic addresses through `ScAddress.String()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/scval.go:ScAddress.String:13-48` — returns `M...` strings for `ScAddressTypeScAddressTypeMuxedAccount`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/strkey/main.go:IsValidEd25519PublicKey:265-275` — accepts only `VersionByteAccountID` (`G...`) values

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestMuxedSACTransferBecomesContractEffect`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestMuxedSACTransferBecomesContractEffect -v`
  4. Observe the test fail because the muxed sender exports as `contract_debited`, with the wrong address and a bogus `details.contract`.

### Test Body

```go
package transform

import (
	"math/big"
	"testing"

	"github.com/stellar/go-stellar-sdk/keypair"
	"github.com/stellar/go-stellar-sdk/support/contractevents"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestMuxedSACTransferBecomesContractEffect(t *testing.T) {
	fromKP := keypair.MustRandom()
	adminKP := keypair.MustRandom()
	toKP := keypair.MustRandom()

	muxedFrom, err := xdr.MuxedAccountFromAccountId(fromKP.Address(), 12345)
	if err != nil {
		t.Fatalf("create muxed account: %v", err)
	}

	tx := makeInvocationTransaction(
		muxedFrom.Address(),
		toKP.Address(),
		adminKP.Address(),
		xdr.MustNewCreditAsset("TEST", adminKP.Address()),
		big.NewInt(10_000_000),
		contractevents.EventTypeTransfer,
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
		t.Fatalf("operation.effects() returned error: %v", err)
	}
	if len(effects) != 2 {
		t.Fatalf("expected 2 effects, got %d", len(effects))
	}

	fromEffect := effects[0]
	if fromEffect.Type != int32(EffectAccountDebited) {
		t.Errorf("expected muxed SAC sender to produce %q effect, got %q (%d)", EffectTypeNames[EffectAccountDebited], fromEffect.TypeString, fromEffect.Type)
	}
	if fromEffect.Address != fromKP.Address() {
		t.Errorf("expected muxed SAC sender address %q, got %q", fromKP.Address(), fromEffect.Address)
	}
	if !fromEffect.AddressMuxed.Valid || fromEffect.AddressMuxed.String != muxedFrom.Address() {
		t.Errorf("expected address_muxed %q, got %+v", muxedFrom.Address(), fromEffect.AddressMuxed)
	}
	if _, ok := fromEffect.Details["contract"]; ok {
		t.Errorf("expected muxed SAC sender to remain an account effect, got contract detail %q", fromEffect.Details["contract"])
	}
	if t.Failed() {
		t.Logf("from effect: %+v", fromEffect)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: A muxed SAC sender is still an account participant, so the exporter should emit `account_debited`, set `address` to the underlying canonical `G...` account, preserve the `M...` value in `address_muxed`, and omit any `contract` detail.
- **Actual**: The exporter emits `contract_debited`, rewrites `address` to the operation source account, leaves `address_muxed` null, and stores the muxed `M...` value in `details.contract`.

## Adversarial Review

1. Exercises claimed bug: YES — the validated PoC uses the existing invoke-host-function test harness and runs the real `transactionOperationWrapper.effects()` path on a transaction whose SAC event contains a muxed `from` endpoint.
2. Realistic preconditions: YES — muxed accounts are a protocol-level `ScAddress` arm, the SDK stringifies them as `M...`, and the effect path consumes those strings exactly as the network would provide them.
3. Bug vs by-design: BUG — the effects layer already preserves muxed identity with `addMuxed()` elsewhere, so treating a muxed account as a contract is an inconsistent classification bug, not an intentional policy.
4. Final severity: High — the exporter silently corrupts effect type, participant identity, and detail semantics for non-financial rows that downstream systems rely on for attribution and filtering.
5. In scope: YES — this is a concrete production transform/export path used by `export_effects`.
6. Test correctness: CORRECT — the test asserts the expected account-effect shape and fails with the actual `contract_debited` row; the observed failure matches the traced source branch exactly.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Recognize muxed SAC endpoints as accounts before falling back to contract effects. One safe approach is to parse the endpoint through `xdr.AddressToMuxedAccount()` or an equivalent helper, then call `addMuxed()` for both classic `G...` and muxed `M...` account participants, reserving the contract branch only for true `C...` contract addresses.
