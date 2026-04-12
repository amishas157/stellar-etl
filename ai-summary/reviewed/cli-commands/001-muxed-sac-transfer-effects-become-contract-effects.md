# H001: Muxed SAC transfer endpoints export as contract effects

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: effect-row type and identity corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_effects` processes a Stellar Asset Contract `transfer` event whose `from` or `to` participant is a muxed account, it should still emit `account_debited` / `account_credited` rows for that participant. The top-level `address` should stay the canonical underlying `G...` account and the muxed `M...` address should be preserved in `address_muxed`, matching the rest of the effect export surface.

## Mechanism

`addInvokeHostFunctionEffects()` decides whether a SAC endpoint is an account by calling `strkey.IsValidEd25519PublicKey(...)`, which only accepts canonical `G...` account strings. But `contractevents.NewStellarAssetContractEvent()` gets its `from` / `to` values from `ScAddress.String()`, and the SDK returns `M...` for muxed account addresses. A valid muxed account therefore falls into the non-account branch and is exported as `contract_debited` / `contract_credited`, with `details.contract="M..."` and the row's top-level `address` incorrectly rebound to the operation source account.

## Trigger

Run `export_effects` on a protocol-23+ ledger containing an `InvokeHostFunction` / SAC `transfer` where either the sender or recipient is a muxed account. Inspect the resulting effect rows: the muxed endpoint should appear as an account effect, but the exporter instead produces a contract effect row keyed to the source account.

## Target Code

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1322-1376` — classifies SAC `transfer` endpoints as account vs contract using `IsValidEd25519PublicKey`
- `github.com/stellar/go-stellar-sdk@v0.1.0/support/contractevents/utils.go:15-51` — parses SAC balance-change addresses through `ScAddress.String()`
- `github.com/stellar/go-stellar-sdk@v0.1.0/xdr/scval.go:13-34` — `ScAddress.String()` returns `M...` for `ScAddressTypeMuxedAccount`

## Evidence

`addInvokeHostFunctionEffects()` has explicit muxed-preserving helpers (`addMuxed`) everywhere else in the file, but the SAC transfer branch bypasses them and treats anything not matching a `G...` key as a contract. The upstream SDK path feeding this function is concrete: `parseBalanceChangeEvent()` stringifies topic addresses, and `ScAddress.String()` constructs a true muxed-account StrKey for muxed participants.

## Anti-Evidence

The surrounding code comments say contract-related rows are intentionally exported as `contract_debited` / `contract_credited`, so pure contract endpoints are expected to follow the contract branch. The viable question is narrower: whether a valid muxed account should ever be treated as a contract at all, and the current `G...`-only predicate suggests it will be.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full data flow from XDR event topics through the SDK's `parseBalanceChangeEvent()` → `ScAddress.String()` → `TransferEvent.From/To` fields, into `addInvokeHostFunctionEffects()` where `strkey.IsValidEd25519PublicKey()` performs a `G...`-only check. The Stellar XDR protocol defines `SC_ADDRESS_TYPE_MUXED_ACCOUNT = 2` as a valid `ScAddress` variant, and the SDK correctly converts it to an `M...` strkey. The effects code's binary account-vs-contract classification has no muxed-account branch, causing three simultaneous corruptions: wrong effect type, wrong address, and wrong detail semantics.

### Code Paths Examined

- `internal/transform/effects.go:1322-1376` — `addInvokeHostFunctionEffects` dispatches on `IsValidEd25519PublicKey()` for all four SAC event types (transfer, mint, clawback, burn); the else branch treats non-`G...` addresses as contracts
- `internal/transform/effects.go:176-198` — `add()` vs `addMuxed()` helper signatures; `addMuxed` correctly extracts the underlying `G...` account and preserves the muxed address, but is only called with `source` (the operation source) in the contract branch, not the actual participant
- `go-stellar-sdk@v0.0.0-20251211085638/support/contractevents/utils.go:20-52` — `parseBalanceChangeEvent` extracts addresses from event topics via `ScAddress.String()`, which produces `M...` for muxed accounts
- `go-stellar-sdk@v0.0.0-20251211085638/xdr/scval.go:13-49` — `ScAddress.String()` has explicit case for `ScAddressTypeScAddressTypeMuxedAccount` (line 24-33), constructing a `MuxedAccount` and calling `GetAddress()` which returns an `M...` strkey
- `go-stellar-sdk@v0.0.0-20251211085638/strkey/main.go:265-276` — `IsValidEd25519PublicKey` decodes with `VersionByteAccountID` only, which rejects `M...` (version byte 12<<3 = 96) since it expects `G...` (version byte 6<<3 = 48)
- `go-stellar-sdk@v0.0.0-20251211085638/xdr/xdr_generated.go:56545-56547` — XDR enum confirms `ScAddressTypeScAddressTypeMuxedAccount = 2` is a protocol-level type

### Findings

**Three simultaneous corruptions when a muxed account appears in a SAC event:**

1. **Effect type corruption**: The row is emitted as `contract_debited`/`contract_credited` (type 96/97) instead of `account_debited`/`account_credited` (type 3/2). Downstream consumers filtering by effect type will either miss the account's balance change or miscount contract activity.

2. **Address identity corruption**: The row's `address` field is set to the *operation source account* (via `e.addMuxed(source, ...)`) instead of the actual participant. If the sender/recipient differs from the source, the effect is attributed to the wrong account entirely.

3. **Detail field semantic corruption**: The `details.contract` field is populated with an `M...` muxed account strkey instead of a `C...` contract address. Any downstream system parsing the `contract` field as a contract ID will get garbage.

**Scope**: All four SAC event types (transfer, mint, clawback, burn) at lines 1354, 1366, 1384, 1402, 1418 share the identical `IsValidEd25519PublicKey` guard and are all affected.

**Triggering conditions**: Requires `ScAddressTypeMuxedAccount` (enum value 2) to appear in SAC event topics, which is defined in the protocol XDR and fully supported by the SDK used by this codebase.

### PoC Guidance

- **Test file**: `internal/transform/effects_test.go`
- **Setup**: Construct a `contractevents.Event` with topic addresses of type `ScAddressTypeScAddressTypeMuxedAccount` (a muxed Ed25519 account). Create a valid SAC transfer event where `From` is a muxed account address. Set up an `effectsWrapper` with a different operation source account.
- **Steps**: Call `addInvokeHostFunctionEffects()` with the crafted event list. Inspect the resulting `e.effects` slice.
- **Assertion**: Assert that the emitted effect has type `EffectAccountDebited` (not `EffectContractDebited`), that `Address` is the underlying `G...` account of the muxed participant (not the source account), that `AddressMuxed` contains the `M...` address, and that `details` does NOT contain a `contract` key.
