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
