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
