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
