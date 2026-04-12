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
