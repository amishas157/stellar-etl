# H002: Muxed SAC mint/burn/clawback endpoints export as contract effects

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: effect-row type and identity corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_effects` processes SAC `mint`, `burn`, or `clawback` events whose credited/debited participant is a muxed account, it should emit the corresponding account effect (`account_credited` for mint, `account_debited` for burn/clawback) for the canonical `G...` account and preserve the `M...` identity in `address_muxed`.

## Mechanism

The `mint`, `burn`, and `clawback` branches reuse the same `strkey.IsValidEd25519PublicKey(...)` account test as the transfer branch. Because the SDK stringifies muxed `ScAddress` values as `M...`, these branches misclassify muxed account recipients/senders as contracts and emit `contract_credited` / `contract_debited` rows instead of account rows. This silently rewrites both the effect type and the account identity for otherwise-valid SAC balance changes.

## Trigger

Run `export_effects` on ledgers containing SAC `mint` to a muxed account, SAC `burn` from a muxed account, or SAC `clawback` from a muxed account. The exported row should target the muxed account holder, but the current code routes the event through the contract-effect branch.

## Target Code

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1380-1428` — `mint`, `clawback`, and `burn` account/contract classification
- `github.com/stellar/go-stellar-sdk@v0.1.0/support/contractevents/utils.go:15-51` — balance-change parser that stringifies `ScAddress` endpoints
- `github.com/stellar/go-stellar-sdk@v0.1.0/xdr/scval.go:13-34` — muxed addresses stringify as `M...`

## Evidence

All three branches have the same structure: test the endpoint with `IsValidEd25519PublicKey`, call `e.add(..., EffectAccount...)` on success, otherwise attach `details["contract"]` and emit `EffectContract...`. That predicate excludes valid muxed account strings even though the parser can hand them back from legitimate on-chain SAC events.

## Anti-Evidence

If the endpoint is a real contract address (`C...`), the contract-effect branch is correct and intentional. This hypothesis depends on a muxed account appearing in the SAC event topics; if the upstream SDK never surfaces muxed participants here, the bug would stay latent.
