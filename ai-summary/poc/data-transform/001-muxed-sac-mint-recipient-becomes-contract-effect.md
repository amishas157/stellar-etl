# H001: Muxed SAC mint recipient becomes a contract effect

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a Stellar Asset Contract `mint` event credits a muxed account, `history_effects` should emit an `account_credited` row for the underlying canonical `G...` account and preserve the muxed `M...` form in `address_muxed`. The effect should stay attributed to the mint recipient, not be rewritten as a contract-side effect on the operation source account.

## Mechanism

The upstream SAC parser preserves muxed recipients as `M...` strings via `ScAddress.String()`. `addInvokeHostFunctionEffects()` then classifies recipients with `strkey.IsValidEd25519PublicKey(...)`, which only accepts canonical `G...` account IDs, so a valid muxed recipient falls into the contract fallback and emits `EffectContractCredited` through `addMuxed(source, ...)` instead of an account credit for the real recipient.

## Trigger

Process an `InvokeHostFunction` transaction whose SAC `mint` event has a muxed `to` address. Export effects and inspect the credited row: it should target the minted account, but the current code will emit `contract_credited`, set the top-level address to the operation source account, and stash the muxed account under `details.contract`.

## Target Code

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1380-1394` — `mint` branch classifies `mintEvent.To` with `IsValidEd25519PublicKey` and falls back to `EffectContractCredited`
- `internal/transform/effects.go:addMuxed:191-197` — existing helper already knows how to preserve canonical `address` plus `address_muxed`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/mint.go:10-40` — parsed mint events expose `To` as a plain string
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/utils.go:10-51` — balance-change parsing converts event addresses through `ScAddress.String()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/scval.go:13-48` — muxed `ScAddress` values stringify as `M...`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/strkey/main.go:265-275` — `IsValidEd25519PublicKey` only accepts `VersionByteAccountID` (`G...`)

## Evidence

The `mint` branch repeats the same account-vs-contract split used for transfer events, but it never recognizes `M...` as an account address. Because the fallback calls `addMuxed(source, EffectContractCredited, ...)`, the exported row will carry the operation source account at the top level and the actual muxed recipient only inside `details["contract"]`.

## Anti-Evidence

Canonical `G...` recipients take the correct `account_credited` branch, and true contract recipients should still produce contract-side effects. The bug requires a live SAC `mint` event whose recipient is a muxed account.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete code path from SAC event parsing through effect generation. The SDK's `parseBalanceChangeEvent` in `utils.go` extracts addresses via `ScAddress.String()`, which produces `M...` strings for `ScAddressTypeScAddressTypeMuxedAccount` (XDR enum value 2). The ETL's `addInvokeHostFunctionEffects` at line 1384 checks `strkey.IsValidEd25519PublicKey(mintEvent.To)`, which only accepts `G...` prefixed addresses. A muxed `M...` address falls into the else branch at line 1391, producing `EffectContractCredited` attributed to the operation source account instead of `EffectAccountCredited` attributed to the actual mint recipient.

### Code Paths Examined

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1322-1394` — SAC event classification switch; mint branch at lines 1380-1394 uses `IsValidEd25519PublicKey` which only accepts `G...`, causing `M...` addresses to fall into the contract fallback
- `internal/transform/effects.go:addMuxed:191-198` — helper correctly decomposes `*xdr.MuxedAccount` into canonical + muxed strings, but is called with `source` (operation source) not the event recipient
- `go-stellar-sdk/support/contractevents/utils.go:parseBalanceChangeEvent` — extracts addresses via `ScAddress.String()`, which returns `M...` for muxed ScAddress types
- `go-stellar-sdk/xdr/scval.go:ScAddress.String()` — for `ScAddressTypeMuxedAccount`, constructs a MuxedAccount and calls `GetAddress()` producing `M...` string
- `go-stellar-sdk/xdr/xdr_generated.go:ScAddressType` — confirms `SC_ADDRESS_TYPE_MUXED_ACCOUNT = 2` is a valid XDR enum value in the current protocol
- `go-stellar-sdk/strkey/main.go:IsValidEd25519PublicKey` — decodes with `VersionByteAccountID` only, rejects `M...` prefixed strings
- `go-stellar-sdk/strkey/main.go:IsValidMuxedAccountEd25519PublicKey` — exists but is NOT called by the ETL code

### Findings

The bug produces three simultaneous data corruptions when a muxed account appears as a SAC mint recipient:

1. **Wrong effect type**: `EffectContractCredited` instead of `EffectAccountCredited` — misclassifies an account operation as a contract operation
2. **Wrong address**: The top-level `address` field is set to the operation source account (via `addMuxed(source, ...)`) instead of the mint recipient
3. **Wrong classification**: The actual muxed recipient is stashed in `details["contract"]` as if it were a contract address

The same pattern affects transfer events (lines 1354-1376) and clawback events (lines 1398-1412), which use identical `IsValidEd25519PublicKey` checks. The SDK provides `strkey.IsValidMuxedAccountEd25519PublicKey()` which could be used but isn't.

The `SC_ADDRESS_TYPE_MUXED_ACCOUNT` type (value 2) is defined in the XDR protocol spec and handled by `ScAddress.String()`, confirming this is a supported address type that can appear in SAC events.

### PoC Guidance

- **Test file**: `internal/transform/effects_test.go`
- **Setup**: Construct a synthetic `contractevents.MintEvent` where `To` is a valid muxed address (`M...` string). This requires building an `xdr.ContractEvent` with `topics[2]` being an `ScVal` wrapping an `ScAddress` of type `ScAddressTypeMuxedAccount`. Use the existing test helpers to create an `operationWrapper` with `InvokeHostFunction` type.
- **Steps**: Call `addInvokeHostFunctionEffects` with the crafted event. Inspect the resulting effects slice.
- **Assertion**: Assert that the emitted effect has type `EffectAccountCredited` (not `EffectContractCredited`), that `Address` equals the canonical `G...` extracted from the muxed account, and that `AddressMuxed` contains the original `M...` string. Currently the code will produce `EffectContractCredited` with the source account's address, demonstrating the bug.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestMuxedSACMintRecipientBecomesContractEffect"
**Test Language**: Go

### Demonstration

The test constructs a SAC mint event where the recipient (`to`) is a valid muxed `M...` address derived from a known `G...` keypair. When the event is processed through `operation.effects()`, the code produces `EffectContractCredited` (type 96) attributed to the admin/source account instead of `EffectAccountCredited` (type 2) attributed to the canonical `G...` of the muxed recipient. The muxed address is incorrectly stashed in `details["contract"]` as if it were a contract address. All three data corruption aspects are confirmed.

### Test Body

```go
package transform

import (
	"fmt"
	"math/big"
	"testing"

	"github.com/stellar/go-stellar-sdk/keypair"
	"github.com/stellar/go-stellar-sdk/strkey"
	"github.com/stellar/go-stellar-sdk/support/contractevents"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/toid"
)

// TestMuxedSACMintRecipientBecomesContractEffect demonstrates that when a SAC
// mint event targets a muxed (M...) account, the effect is misclassified as
// EffectContractCredited on the operation source instead of EffectAccountCredited
// on the actual mint recipient.
//
// Bug: strkey.IsValidEd25519PublicKey only accepts G... addresses. A valid M...
// muxed address falls into the else branch, producing EffectContractCredited
// attributed to the operation source account.
func TestMuxedSACMintRecipientBecomesContractEffect(t *testing.T) {
	// 1. Construct a muxed M... address from a known G... keypair
	recipientKP := keypair.MustRandom()
	recipientG := recipientKP.Address() // canonical G... address

	muxedAcct := &strkey.MuxedAccount{}
	if err := muxedAcct.SetAccountID(recipientG); err != nil {
		t.Fatalf("failed to set account ID on muxed account: %v", err)
	}
	muxedAcct.SetID(12345) // arbitrary muxed ID

	muxedM, err := muxedAcct.Address()
	if err != nil {
		t.Fatalf("failed to get muxed address: %v", err)
	}

	// Sanity: the M address should NOT pass IsValidEd25519PublicKey
	if strkey.IsValidEd25519PublicKey(muxedM) {
		t.Fatal("M... address should not be valid as Ed25519 public key")
	}
	// But it IS a valid muxed account
	if !strkey.IsValidMuxedAccountEd25519PublicKey(muxedM) {
		t.Fatalf("M... address should be valid muxed account: %s", muxedM)
	}

	// 2. Build a mint event with the muxed address as the recipient
	admin := keypair.MustRandom().Address()
	asset := xdr.MustNewCreditAsset("TESTER", admin)
	amount := big.NewInt(12345)

	tx := makeInvocationTransaction(
		"" /* from - unused for mint */, muxedM,
		admin,
		asset,
		amount,
		contractevents.EventTypeMint,
	)

	if !tx.Result.Successful() {
		t.Fatal("transaction should be successful")
	}

	operation := transactionOperationWrapper{
		index:          0,
		transaction:    tx,
		operation:      tx.Envelope.Operations()[0],
		ledgerSequence: 1,
		network:        networkPassphrase,
	}

	// 3. Generate effects
	effects, err := operation.effects()
	if err != nil {
		t.Fatalf("effects() returned error: %v", err)
	}

	if len(effects) != 1 {
		t.Fatalf("expected exactly 1 effect, got %d", len(effects))
	}

	effect := effects[0]
	expectedOpID := toid.New(1, 0, 1).ToInt64()

	// 4. Assert: the effect should be EffectAccountCredited for the recipient.
	//    BUG: the code actually produces EffectContractCredited on the admin.

	// Check the effect type — this reveals the misclassification
	if effect.Type == int32(EffectContractCredited) {
		t.Errorf("BUG CONFIRMED: effect type is EffectContractCredited (%d), "+
			"expected EffectAccountCredited (%d). "+
			"Muxed mint recipient was misclassified as a contract.",
			EffectContractCredited, EffectAccountCredited)
	} else if effect.Type == int32(EffectAccountCredited) {
		t.Logf("Effect type is correctly EffectAccountCredited")
	} else {
		t.Errorf("unexpected effect type: %d (%s)", effect.Type, effect.TypeString)
	}

	// Check the address — should be the canonical G... of the muxed recipient
	if effect.Address != recipientG {
		t.Errorf("BUG CONFIRMED: effect address is %s (the admin/source), "+
			"expected %s (the canonical G... of the muxed mint recipient)",
			effect.Address, recipientG)
	}

	// Check that the muxed recipient was stashed as details["contract"]
	// instead of being preserved as the proper address
	if contractVal, ok := effect.Details["contract"]; ok {
		t.Errorf("BUG CONFIRMED: muxed recipient was stashed in details[\"contract\"] = %v "+
			"instead of being the primary effect address", contractVal)
	}

	// Log full effect for diagnostic purposes
	t.Logf("Effect: Type=%d (%s), Address=%s, AddressMuxed=%s, OperationID=%d, Details=%v",
		effect.Type, effect.TypeString, effect.Address,
		effect.AddressMuxed.String, effect.OperationID, effect.Details)

	// Summary assertion: if any of the three bugs are present, fail definitively
	isBuggy := effect.Type == int32(EffectContractCredited) ||
		effect.Address != recipientG ||
		effect.Details["contract"] != nil

	if isBuggy {
		t.Errorf("SUMMARY: Muxed SAC mint recipient produces wrong effect. "+
			"Got: type=%s, address=%s (admin=%s, expected=%s). "+
			"Expected: type=%s, address=%s, address_muxed=%s",
			EffectTypeNames[EffectType(effect.Type)], effect.Address, admin, recipientG,
			EffectTypeNames[EffectAccountCredited], recipientG, muxedM)
	}

	_ = expectedOpID
	_ = fmt.Sprintf // avoid unused import
}
```

### Test Output

```
=== RUN   TestMuxedSACMintRecipientBecomesContractEffect
    data_integrity_poc_test.go:91: BUG CONFIRMED: effect type is EffectContractCredited (96), expected EffectAccountCredited (2). Muxed mint recipient was misclassified as a contract.
    data_integrity_poc_test.go:103: BUG CONFIRMED: effect address is GD5KJO5B2MMTQG5EVOT5YGVIORZVRDJTU4WZRMMRIUOAOBBRSU2XAFTO (the admin/source), expected GD5MFPAEEZGXARLBDB7A5VXRIUCDWPKTKDV3HIT5ZXV6PCZH7EPNXMN7 (the canonical G... of the muxed mint recipient)
    data_integrity_poc_test.go:111: BUG CONFIRMED: muxed recipient was stashed in details["contract"] = MD5MFPAEEZGXARLBDB7A5VXRIUCDWPKTKDV3HIT5ZXV6PCZH7EPNWAAAAAAAAABQHGREQ instead of being the primary effect address
    data_integrity_poc_test.go:116: Effect: Type=96 (contract_credited), Address=GD5KJO5B2MMTQG5EVOT5YGVIORZVRDJTU4WZRMMRIUOAOBBRSU2XAFTO, AddressMuxed=, OperationID=4294967297, Details=map[amount:0.0012345 asset_code:TESTER asset_issuer:GD5KJO5B2MMTQG5EVOT5YGVIORZVRDJTU4WZRMMRIUOAOBBRSU2XAFTO asset_type:credit_alphanum12 contract:MD5MFPAEEZGXARLBDB7A5VXRIUCDWPKTKDV3HIT5ZXV6PCZH7EPNWAAAAAAAAABQHGREQ contract_event_type:mint]
    data_integrity_poc_test.go:126: SUMMARY: Muxed SAC mint recipient produces wrong effect. Got: type=contract_credited, address=GD5KJO5B2MMTQG5EVOT5YGVIORZVRDJTU4WZRMMRIUOAOBBRSU2XAFTO (admin=GD5KJO5B2MMTQG5EVOT5YGVIORZVRDJTU4WZRMMRIUOAOBBRSU2XAFTO, expected=GD5MFPAEEZGXARLBDB7A5VXRIUCDWPKTKDV3HIT5ZXV6PCZH7EPNXMN7). Expected: type=account_credited, address=GD5MFPAEEZGXARLBDB7A5VXRIUCDWPKTKDV3HIT5ZXV6PCZH7EPNXMN7, address_muxed=MD5MFPAEEZGXARLBDB7A5VXRIUCDWPKTKDV3HIT5ZXV6PCZH7EPNWAAAAAAAAABQHGREQ
--- FAIL: TestMuxedSACMintRecipientBecomesContractEffect (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.785s
```
