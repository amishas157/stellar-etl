# H002: Muxed SAC clawback sender becomes a contract effect

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a Stellar Asset Contract `clawback` event debits a muxed account, `history_effects` should emit an `account_debited` row for the underlying canonical `G...` account and preserve the muxed `M...` value in `address_muxed`. The debited participant should remain the clawed-back holder, not be rewritten as a contract pseudo-address on the operation source account.

## Mechanism

`contractevents.ClawbackEvent` parses the debited holder from an `ScAddress` string, so muxed holders arrive as `M...`. The `clawback` branch in `addInvokeHostFunctionEffects()` again uses `strkey.IsValidEd25519PublicKey(...)` as its account test; muxed holders fail that check and are misrouted into the contract fallback, which emits `EffectContractDebited` for the source account and stores the real muxed holder under `details["contract"]`.

## Trigger

Process an `InvokeHostFunction` transaction whose SAC `clawback` event has a muxed `from` address. Export effects and inspect the debited row: instead of an account debit on the holder with preserved muxed metadata, the current code should emit `contract_debited` on the operation source account.

## Target Code

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1396-1412` — `clawback` branch classifies `cbEvent.From` with `IsValidEd25519PublicKey` and falls back to `EffectContractDebited`
- `internal/transform/effects.go:addMuxed:191-197` — correct muxed-account projection already exists elsewhere in the effects layer
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/clawback.go:10-40` — parsed clawback events expose `From` as a string
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/utils.go:10-51` — balance-change parsing converts event addresses through `ScAddress.String()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/scval.go:13-48` — muxed `ScAddress` values stringify as `M...`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/strkey/main.go:265-275` — `IsValidEd25519PublicKey` rejects muxed account strings

## Evidence

The `clawback` branch is a copy of the already-proven transfer classification pattern, just on a different event type. If `cbEvent.From` is muxed, the code never attempts to decode it as an account and instead emits a contract debit attributed to the wrong top-level address.

## Anti-Evidence

Ordinary `G...` clawback holders are exported correctly, and true contract addresses should still remain contract-side effects. The corruption only appears when the clawed-back holder is a muxed account.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from XDR definition through SDK parsing to the effects classification logic. The `SCAddress` XDR union explicitly includes `SC_ADDRESS_TYPE_MUXED_ACCOUNT = 2` (Stellar-contract.x). When a SAC clawback event contains a muxed `ScAddress`, `parseBalanceChangeEvent()` calls `ScAddress.String()` which constructs a `MuxedAccount` and returns an `M...` strkey. The clawback branch at line 1402 tests this with `strkey.IsValidEd25519PublicKey()`, which only accepts `G...` addresses (it decodes with `VersionByteAccountID`). The muxed `M...` address fails and falls through to the else branch, which misattributes the effect.

### Code Paths Examined

- `xdr/Stellar-contract.x:SCAddress` — XDR union defines `SC_ADDRESS_TYPE_MUXED_ACCOUNT = 2` with `MuxedEd25519Account` payload
- `xdr/scval.go:ScAddress.String:24-33` — For muxed type, constructs `MuxedAccount` with `CryptoKeyTypeKeyTypeMuxedEd25519` and calls `GetAddress()`, producing `M...` strkey
- `support/contractevents/utils.go:parseBalanceChangeEvent:26-34` — Calls `topics[1].GetAddress()` then `firstSc.String()`, so muxed addresses become `M...` strings
- `support/contractevents/clawback.go:parse:36` — Delegates to `parseBalanceChangeEvent`, stores result as `ClawbackEvent.From` (string)
- `strkey/main.go:IsValidEd25519PublicKey:266-276` — Decodes with `VersionByteAccountID` only; `M...` addresses fail (wrong version byte)
- `internal/transform/effects.go:1396-1412` — Clawback branch: `IsValidEd25519PublicKey(cbEvent.From)` returns false for muxed, falls to else: sets `details["contract"] = cbEvent.From`, emits `EffectContractDebited` on `source` (operation source account)
- `internal/transform/effects.go:addMuxed:191-197` — Correct muxed projection exists (extracts G... + sets address_muxed) but is not used for the clawed-back holder

### Findings

The bug is confirmed through complete code path tracing:

1. **XDR support is real**: `SC_ADDRESS_TYPE_MUXED_ACCOUNT` is value 2 in the `SCAddressType` enum, defined in `Stellar-contract.x`. The SDK fully handles this type in `ScAddress.String()`.

2. **Address classification is binary and wrong**: The code uses `IsValidEd25519PublicKey` as a two-way gate: either it's a `G...` account or it's treated as a contract. There is no third check for muxed `M...` addresses (e.g., `IsValidMuxedAccountEd25519PublicKey` at strkey/main.go:305-308 exists but is unused here).

3. **Three-way corruption**: When the clawed-back holder is muxed:
   - **Wrong effect type**: `EffectContractDebited` instead of `EffectAccountDebited`
   - **Wrong account**: Effect is attributed to the operation source account instead of the clawed-back holder
   - **Wrong metadata**: The actual `M...` address is buried in `details["contract"]` instead of being decoded into the standard `address` + `address_muxed` fields

4. **Same pattern affects all four SAC event types**: Transfer (line 1354), mint (line 1384), clawback (line 1402), and burn (line 1418) all use the identical `IsValidEd25519PublicKey` gate. A sibling hypothesis (001) already covers mint; this one covers clawback.

5. **Fix path is clear**: `strkey.IsValidMuxedAccountEd25519PublicKey` already exists in the SDK (strkey/main.go:305-308). The clawback branch should check for both `G...` and `M...` addresses before falling through to the contract path. For `M...` addresses, it should decode the underlying `G...` key and muxed ID, then emit `EffectAccountDebited` with proper `address_muxed` metadata.

### PoC Guidance

- **Test file**: `internal/transform/effects_test.go` (append to existing SAC event effect tests)
- **Setup**: Construct an `InvokeHostFunction` operation with a SAC clawback event where the `From` topic is an `ScAddress` of type `ScAddressTypeMuxedAccount`. Use a known Ed25519 public key with a muxed ID (e.g., id=42) to construct the `MuxedEd25519Account` payload.
- **Steps**: Call `addInvokeHostFunctionEffects()` with the constructed event list. Inspect the resulting effects.
- **Assertion**: Assert that the effect type is `EffectAccountDebited` (not `EffectContractDebited`), that the account address is the underlying `G...` address (not the operation source), and that `address_muxed` contains the full `M...` strkey. Also assert that `details["contract"]` is NOT set.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestMuxedSACClawbackSenderBecomesContractEffect"
**Test Language**: Go

### Demonstration

The test constructs a SAC clawback event where the debited holder (`from`) is a muxed M... address (created from a known G... key with muxed ID=42). When processed through the production `effects()` code path, the resulting effect has type `EffectContractDebited` (97) instead of `EffectAccountDebited` (3), is attributed to the admin/source account instead of the underlying G... account, and the real M... address is buried in `details["contract"]` rather than being decoded into proper `address` + `address_muxed` fields. All three corruption signals are confirmed in a single test run.

### Test Body

```go
//nolint:typecheck
package transform

import (
	"fmt"
	"math/big"
	"strings"
	"testing"

	"github.com/stellar/go-stellar-sdk/keypair"
	"github.com/stellar/go-stellar-sdk/support/contractevents"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestMuxedSACClawbackSenderBecomesContractEffect demonstrates that when a SAC
// clawback event debits a muxed account (M... address), the effect is misclassified
// as EffectContractDebited on the operation source account, instead of
// EffectAccountDebited on the underlying G... account with muxed metadata.
func TestMuxedSACClawbackSenderBecomesContractEffect(t *testing.T) {
	// 1. Construct a muxed account address from a known G... key
	underlyingKP := keypair.MustRandom()
	underlyingG := underlyingKP.Address() // G... address

	muxedAcct, err := xdr.MuxedAccountFromAccountId(underlyingG, 42)
	if err != nil {
		t.Fatalf("failed to create muxed account: %v", err)
	}
	muxedM := muxedAcct.Address() // M... address

	// Sanity: the M address should start with 'M'
	if !strings.HasPrefix(muxedM, "M") {
		t.Fatalf("expected muxed address to start with M, got %s", muxedM)
	}

	admin := keypair.MustRandom().Address()
	asset := xdr.MustNewCreditAsset("TESTER", admin)
	amount := big.NewInt(12345)

	// 2. Build an InvokeHostFunction transaction with a SAC clawback event
	//    where the "from" participant is the muxed M... address.
	tx := makeInvocationTransaction(
		muxedM, "" /* to is unused for clawback */, admin,
		asset, amount,
		contractevents.EventTypeClawback,
	)

	operation := transactionOperationWrapper{
		index:          0,
		transaction:    tx,
		operation:      tx.Envelope.Operations()[0],
		ledgerSequence: 1,
		network:        networkPassphrase,
	}

	// 3. Run the production code path
	effects, err := operation.effects()
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}

	if len(effects) == 0 {
		t.Fatal("expected at least one effect, got none")
	}

	effect := effects[0]

	// 4. Assert the output is wrong — proving the bug.
	//
	// CORRECT behavior would be:
	//   - Type = EffectAccountDebited (3)
	//   - Address = underlying G... address
	//   - No "contract" key in details
	//
	// ACTUAL (buggy) behavior:
	//   - Type = EffectContractDebited (97)
	//   - Address = admin (operation source), NOT the clawed-back holder
	//   - details["contract"] = the M... address

	// The effect type is EffectContractDebited instead of EffectAccountDebited
	if effect.Type != int32(EffectContractDebited) {
		t.Errorf("BUG NOT REPRODUCED: expected effect type EffectContractDebited (%d) due to muxed misclassification, got %d (%s)",
			EffectContractDebited, effect.Type, effect.TypeString)
	} else {
		t.Logf("CONFIRMED: muxed clawback holder produced EffectContractDebited (%d) instead of EffectAccountDebited (%d)",
			EffectContractDebited, EffectAccountDebited)
	}

	// The effect is attributed to the admin (source account) instead of the
	// underlying G... account of the muxed holder
	if effect.Address != admin {
		t.Errorf("BUG NOT REPRODUCED: expected effect address to be admin %s (source misattribution), got %s",
			admin, effect.Address)
	} else {
		t.Logf("CONFIRMED: effect address is admin (source) %s instead of underlying account %s",
			admin, underlyingG)
	}

	// The real muxed address is buried in details["contract"] instead of being
	// properly decoded into address + address_muxed fields
	contractVal, hasContract := effect.Details["contract"]
	if !hasContract {
		t.Error("BUG NOT REPRODUCED: expected details[\"contract\"] to contain the muxed address")
	} else {
		t.Logf("CONFIRMED: details[\"contract\"] = %v (should not exist; muxed addr should be in address_muxed)", contractVal)
	}

	// Summary assertion: if all three corruption signals are present, the bug is proven
	if effect.Type == int32(EffectContractDebited) && effect.Address == admin && hasContract {
		t.Logf("=== BUG PROVEN: muxed SAC clawback from=%s produces wrong effect type (%s), "+
			"wrong account (%s instead of %s), and misplaced address in details[\"contract\"] ===",
			muxedM, EffectTypeNames[EffectContractDebited], admin, underlyingG)
	}

	_ = fmt.Sprintf("effect: %+v", effect) // prevent unused import
}
```

### Test Output

```
=== RUN   TestMuxedSACClawbackSenderBecomesContractEffect
    data_integrity_poc_test.go:84: CONFIRMED: muxed clawback holder produced EffectContractDebited (97) instead of EffectAccountDebited (3)
    data_integrity_poc_test.go:94: CONFIRMED: effect address is admin (source) GASB6WZBD2TV7IV7T3PWW3PZWJDLL2C3SJH7OSRASAQECQOVMRA6IAXY instead of underlying account GARYMJFGUGYENSIOTAJUYWLH4BCL4Y654KNQTFWNFICFBNNGTVEBBPRI
    data_integrity_poc_test.go:104: CONFIRMED: details["contract"] = MARYMJFGUGYENSIOTAJUYWLH4BCL4Y654KNQTFWNFICFBNNGTVEBAAAAAAAAAAAAFKQ6E (should not exist; muxed addr should be in address_muxed)
    data_integrity_poc_test.go:109: === BUG PROVEN: muxed SAC clawback from=MARYMJFGUGYENSIOTAJUYWLH4BCL4Y654KNQTFWNFICFBNNGTVEBAAAAAAAAAAAAFKQ6E produces wrong effect type (contract_debited), wrong account (GASB6WZBD2TV7IV7T3PWW3PZWJDLL2C3SJH7OSRASAQECQOVMRA6IAXY instead of GARYMJFGUGYENSIOTAJUYWLH4BCL4Y654KNQTFWNFICFBNNGTVEBBPRI), and misplaced address in details["contract"] ===
--- PASS: TestMuxedSACClawbackSenderBecomesContractEffect (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.753s
```
