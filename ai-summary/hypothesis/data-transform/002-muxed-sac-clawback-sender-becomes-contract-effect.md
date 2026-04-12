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
