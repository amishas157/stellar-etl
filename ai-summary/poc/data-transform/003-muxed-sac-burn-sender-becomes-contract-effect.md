# H003: Muxed SAC burn sender becomes a contract effect

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a Stellar Asset Contract `burn` event debits a muxed account, `history_effects` should export an `account_debited` row for the underlying canonical `G...` account and preserve the original muxed `M...` address in `address_muxed`. The effect should represent the burned holder, not a fabricated contract debit on the operation source account.

## Mechanism

`contractevents.BurnEvent` reads the burn source from `ScAddress.String()`, which returns `M...` for muxed accounts. The `burn` branch then applies the same `IsValidEd25519PublicKey(...)` test that only recognizes `G...`, so muxed holders fall through to the contract branch and are emitted as `EffectContractDebited` rows whose top-level address is rebound to the operation source account.

## Trigger

Process an `InvokeHostFunction` transaction whose SAC `burn` event has a muxed `from` address. Export effects and inspect the debited row: it should be an account debit with muxed metadata, but the current code should instead emit `contract_debited` with `details.contract` holding the muxed address.

## Target Code

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1414-1428` — `burn` branch classifies `burnEvent.From` with `IsValidEd25519PublicKey` and falls back to `EffectContractDebited`
- `internal/transform/effects.go:addMuxed:191-197` — reusable helper for correct `address` / `address_muxed` emission
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/burn.go:10-56` — parsed burn events expose `From` as a string derived from `ScAddress`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/scval.go:13-48` — muxed `ScAddress` values stringify as `M...`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/strkey/main.go:265-275` — `IsValidEd25519PublicKey` only validates canonical account IDs

## Evidence

Unlike transfer/mint/clawback, the burn parser has its own file, but it still converts the participant through `ScAddress.String()` and hands `M...` straight to the same narrow validator. The fallback branch then misclassifies the participant as a contract and exports the wrong effect type and wrong top-level address.

## Anti-Evidence

Canonical `G...` burn senders still export correctly, and true contract burn participants should remain contract effects. The bad output depends on a real muxed burn source in the SAC event.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete code path from SAC burn event parsing through effect classification. The SDK's `BurnEvent.parse()` in `burn.go` extracts `From` via `topics[1].GetAddress()` → `ScAddress.String()`, which returns `M...` for `ScAddressTypeScAddressTypeMuxedAccount` (lines 24-33 of scval.go). The ETL's burn branch at effects.go line 1418 checks `strkey.IsValidEd25519PublicKey(burnEvent.From)`, which only accepts `G...` addresses (decodes with `VersionByteAccountID`). A muxed `M...` address falls into the else branch at line 1425, producing `EffectContractDebited` attributed to the operation source account instead of `EffectAccountDebited` attributed to the actual burn holder.

### Code Paths Examined

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1414-1428` — SAC burn branch uses `IsValidEd25519PublicKey(burnEvent.From)` at line 1418, which rejects `M...` addresses; fallback at line 1425-1427 sets `details["contract"] = burnEvent.From` and calls `e.addMuxed(source, EffectContractDebited, details)` using the operation source account
- `internal/transform/effects.go:addMuxed:191-198` — helper correctly decomposes `*xdr.MuxedAccount` into canonical + muxed strings, but is called with `source` (operation source account), not the event participant
- `go-stellar-sdk/support/contractevents/burn.go:parse:23-57` — `BurnEvent` parses 3-topic events; `From` comes from `topics[1].GetAddress()` → `ScAddress.String()`, so muxed addresses arrive as `M...` strings
- `go-stellar-sdk/xdr/scval.go:ScAddress.String:24-33` — For `ScAddressTypeMuxedAccount`, constructs a `MuxedAccount` with `CryptoKeyTypeKeyTypeMuxedEd25519` and calls `GetAddress()`, producing `M...` strkey
- `go-stellar-sdk/strkey/main.go:IsValidEd25519PublicKey:266` — Decodes with `VersionByteAccountID` only; `M...` addresses fail validation
- `go-stellar-sdk/strkey/main.go:IsValidMuxedAccountEd25519PublicKey:306` — Exists in the SDK but is NOT called anywhere in the ETL burn path

### Findings

The bug produces three simultaneous data corruptions when a muxed account appears as a SAC burn source:

1. **Wrong effect type**: `EffectContractDebited` instead of `EffectAccountDebited` — misclassifies an account debit as a contract operation
2. **Wrong address**: The top-level `address` field is set to the operation source account (via `addMuxed(source, ...)`) instead of the burn holder
3. **Wrong metadata**: The actual muxed `M...` address is stored in `details["contract"]` as if it were a contract address, instead of being decoded into the standard `address` + `address_muxed` fields

Note that the burn event has a distinct parser (`burn.go` handles 3-topic events) compared to transfer/mint/clawback (which use the 4-topic `parseBalanceChangeEvent`). However, the address extraction logic is identical — `topics[1].GetAddress()` → `ScAddress.String()` — and the ETL classification code uses the same flawed `IsValidEd25519PublicKey` gate.

The `SC_ADDRESS_TYPE_MUXED_ACCOUNT` type (value 2) is defined in the XDR protocol and fully handled by `ScAddress.String()`, confirming muxed addresses can legitimately appear in SAC burn events.

### PoC Guidance

- **Test file**: `internal/transform/effects_test.go`
- **Setup**: Construct a synthetic `contractevents.BurnEvent` where `From` is a valid muxed address (`M...` string). This requires building an `xdr.ContractEvent` with 3 topics: topic[0] = "burn" symbol, topic[1] = `ScVal` wrapping an `ScAddress` of type `ScAddressTypeMuxedAccount`, topic[2] = asset bytes. Use the existing test helpers to create an `operationWrapper` with `InvokeHostFunction` type.
- **Steps**: Call `addInvokeHostFunctionEffects` with the crafted event. Inspect the resulting effects slice.
- **Assertion**: Assert that the emitted effect has type `EffectAccountDebited` (not `EffectContractDebited`), that `Address` equals the canonical `G...` extracted from the muxed account, and that `AddressMuxed` contains the original `M...` string. Currently the code will produce `EffectContractDebited` with the source account's address, demonstrating the bug.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestMuxedSACBurnSenderBecomesContractEffect"
**Test Language**: Go

### Demonstration

The test constructs a SAC burn event with a muxed `M...` from address and runs it through the production `effects()` code path. The test confirms all three simultaneous data corruptions: (1) the effect type is `EffectContractDebited` instead of `EffectAccountDebited`, (2) the address is set to the operation source account (admin) instead of the canonical `G...` address embedded in the muxed account, and (3) the muxed `M...` address is stored in `details["contract"]` as if it were a contract address rather than being properly decomposed into `address`/`address_muxed` fields.

### Test Body

```go
//nolint:typecheck
package transform

import (
	"fmt"
	"math/big"
	"strings"
	"testing"
	"time"

	"github.com/stellar/go-stellar-sdk/keypair"
	"github.com/stellar/go-stellar-sdk/strkey"
	"github.com/stellar/go-stellar-sdk/support/contractevents"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/toid"
)

// TestMuxedSACBurnSenderBecomesContractEffect demonstrates that when a SAC burn
// event has a muxed (M...) from address, the ETL misclassifies it as
// EffectContractDebited attributed to the operation source account, instead of
// EffectAccountDebited attributed to the canonical G... account embedded in
// the muxed address.
//
// Root cause: effects.go burn branch uses strkey.IsValidEd25519PublicKey which
// only accepts G... addresses; M... fails the check and falls through to the
// contract-effect branch.
func TestMuxedSACBurnSenderBecomesContractEffect(t *testing.T) {
	// 1. Build addresses
	kp := keypair.MustRandom()
	canonicalAddr := kp.Address() // G... address

	// Create a muxed M... address from the same keypair with muxed ID = 42
	muxedAcct := &strkey.MuxedAccount{}
	if err := muxedAcct.SetAccountID(canonicalAddr); err != nil {
		t.Fatalf("failed to set account id: %v", err)
	}
	muxedAcct.SetID(42)
	muxedAddr, err := muxedAcct.Address()
	if err != nil {
		t.Fatalf("failed to get muxed address: %v", err)
	}

	// Verify: M... address is NOT a valid Ed25519 public key per strkey
	if strkey.IsValidEd25519PublicKey(muxedAddr) {
		t.Fatal("muxed address should not pass IsValidEd25519PublicKey")
	}

	admin := keypair.MustRandom().Address()
	asset := xdr.MustNewCreditAsset("TESTER", admin)
	amount := big.NewInt(12345)

	// 2. Create the transaction with a burn event where from = muxed address
	tx := makeInvocationTransaction(
		muxedAddr, "",
		admin,
		asset,
		amount,
		contractevents.EventTypeBurn,
	)

	operation := transactionOperationWrapper{
		index:          0,
		transaction:    tx,
		operation:      tx.Envelope.Operations()[0],
		ledgerSequence: 1,
		network:        networkPassphrase,
	}

	// 3. Run effects generation
	effects, err := operation.effects()
	if err != nil {
		t.Fatalf("unexpected error generating effects: %v", err)
	}

	if len(effects) != 1 {
		t.Fatalf("expected exactly 1 effect, got %d", len(effects))
	}

	effect := effects[0]

	// 4. Demonstrate the bug:
	// The correct behavior would be:
	//   - Type = EffectAccountDebited
	//   - Address = canonical G... address of the muxed account
	// But the actual behavior is:
	//   - Type = EffectContractDebited (wrong)
	//   - Address = admin (operation source account, wrong)
	//   - details["contract"] = muxed M... address (wrong)

	// Assert the bug exists: effect type is EffectContractDebited instead of EffectAccountDebited
	if effect.Type != int32(EffectContractDebited) {
		t.Errorf("BUG NOT DEMONSTRATED: expected effect type EffectContractDebited (%d) due to bug, got %s (%d)",
			EffectContractDebited, effect.TypeString, effect.Type)
	} else {
		t.Logf("BUG CONFIRMED: effect type is EffectContractDebited (%d) instead of correct EffectAccountDebited (%d)",
			effect.Type, EffectAccountDebited)
	}

	// Assert the bug: address is the admin (op source), not the canonical G... from the muxed account
	if effect.Address != canonicalAddr && effect.Address == admin {
		t.Logf("BUG CONFIRMED: address is admin %s instead of canonical account %s", admin, canonicalAddr)
	} else if effect.Address == canonicalAddr {
		t.Errorf("BUG NOT DEMONSTRATED: address is correctly set to canonical account %s", canonicalAddr)
	}

	// Assert the bug: details["contract"] is set to the muxed M... address
	contract, hasContract := effect.Details["contract"]
	if hasContract && contract == muxedAddr {
		t.Logf("BUG CONFIRMED: details[\"contract\"] = %s (muxed address treated as contract)", muxedAddr)
	} else if !hasContract {
		t.Errorf("BUG NOT DEMONSTRATED: details[\"contract\"] not present")
	}

	// Summary: verify all three corruptions simultaneously
	expectedBuggyEffect := EffectOutput{
		Address:     admin,
		OperationID: toid.New(1, 0, 1).ToInt64(),
		Details: map[string]interface{}{
			"amount":              "0.0012345",
			"asset_code":          strings.Trim(asset.GetCode(), "\x00"),
			"asset_issuer":        asset.GetIssuer(),
			"asset_type":          "credit_alphanum12",
			"contract":            muxedAddr,
			"contract_event_type": "burn",
		},
		Type:           int32(EffectContractDebited),
		TypeString:     EffectTypeNames[EffectContractDebited],
		LedgerClosed:   time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
		LedgerSequence: 1,
		EffectIndex:    0,
		EffectId:       fmt.Sprintf("%d-%d", toid.New(1, 0, 1).ToInt64(), 0),
	}

	if effect.Type == expectedBuggyEffect.Type &&
		effect.Address == expectedBuggyEffect.Address &&
		effect.Details["contract"] == expectedBuggyEffect.Details["contract"] {
		t.Logf("ALL THREE CORRUPTIONS CONFIRMED:")
		t.Logf("  1. Wrong effect type: EffectContractDebited instead of EffectAccountDebited")
		t.Logf("  2. Wrong address: %s (admin) instead of %s (canonical account)", admin, canonicalAddr)
		t.Logf("  3. Wrong metadata: details[\"contract\"]=%s instead of address/address_muxed fields", muxedAddr)
	}
}
```

### Test Output

```
=== RUN   TestMuxedSACBurnSenderBecomesContractEffect
    data_integrity_poc_test.go:95: BUG CONFIRMED: effect type is EffectContractDebited (97) instead of correct EffectAccountDebited (3)
    data_integrity_poc_test.go:101: BUG CONFIRMED: address is admin GAO5WGG42SNRVIARC4EQUJ2VH6NUR3OW5P4RAOY62WOUUMVIGPWIU7S7 instead of canonical account GAGRTA62CPIF3HT7DB67XIL3NZAKUMUEB7VKE45OTU7QWYRIVVDAYE6A
    data_integrity_poc_test.go:109: BUG CONFIRMED: details["contract"] = MAGRTA62CPIF3HT7DB67XIL3NZAKUMUEB7VKE45OTU7QWYRIVVDAYAAAAAAAAAAAFJD2G (muxed address treated as contract)
    data_integrity_poc_test.go:137: ALL THREE CORRUPTIONS CONFIRMED:
    data_integrity_poc_test.go:138:   1. Wrong effect type: EffectContractDebited instead of EffectAccountDebited
    data_integrity_poc_test.go:139:   2. Wrong address: GAO5WGG42SNRVIARC4EQUJ2VH6NUR3OW5P4RAOY62WOUUMVIGPWIU7S7 (admin) instead of GAGRTA62CPIF3HT7DB67XIL3NZAKUMUEB7VKE45OTU7QWYRIVVDAYE6A (canonical account)
    data_integrity_poc_test.go:140:   3. Wrong metadata: details["contract"]=MAGRTA62CPIF3HT7DB67XIL3NZAKUMUEB7VKE45OTU7QWYRIVVDAYAAAAAAAAAAAFJD2G instead of address/address_muxed fields
--- PASS: TestMuxedSACBurnSenderBecomesContractEffect (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.752s
```
