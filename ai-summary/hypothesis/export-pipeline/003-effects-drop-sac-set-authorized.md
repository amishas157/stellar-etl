# H003: `export_effects` silently drops Stellar Asset Contract `set_authorized` events

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A successful Stellar Asset Contract authorization change should surface in the
normalized effect stream, especially when the same transaction already exports a
raw contract-event row for that state transition. Downstream consumers should
not have to infer SAC authorization changes from `history_contract_events` while
`history_effects` stays empty for the same ledger range.

## Mechanism

`addInvokeHostFunctionEffects()` feeds every Soroban contract event through
`contractevents.NewStellarAssetContractEvent()` and silently `continue`s on any
error. The SAC helper already reserves `EventTypeSetAuthorized` as a valid event
class, but its topic map only recognizes `transfer`, `mint`, `clawback`, and
`burn`, so a real `set_authorized` SAC event is treated as unsupported and
discarded before any effect row can be emitted.

## Trigger

1. Execute a Stellar Asset Contract `set_authorized` call that emits a SAC
   authorization event in a ledger range.
2. Export both:
   1. `stellar-etl export_contract_events --start-ledger <L> --end-ledger <R>`
   2. `stellar-etl export_effects --start-ledger <L> --end-ledger <R>`
3. The contract-event export should contain the raw authorization event, but the
   effect export will emit no corresponding row because the helper rejects the
   event topic and `addInvokeHostFunctionEffects()` drops it.

## Target Code

- `internal/transform/effects.go:1322-1430` — invoke-host-function effects discard any unsupported SAC event with `continue`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/event.go:17-39` — SAC event type enum reserves `EventTypeSetAuthorized`, but the topic map omits it
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/event.go:61-90` — `NewStellarAssetContractEvent()` rejects any topic not present in `STELLAR_ASSET_CONTRACT_TOPICS`
- `internal/transform/contract_events.go:21-67` — raw contract-event export still writes the underlying event stream for the same transaction

## Evidence

The support library explicitly distinguishes between implemented SAC event types
and TODO event types, and `set_authorized` is listed in the latter bucket. The
effects transformer has no fallback path for those events: every parser error is
classified as "irrelevant or unsupported" and skipped. That makes SAC
authorization changes disappear from `history_effects` even though the same
ledger range can still expose them through `history_contract_events`.

## Anti-Evidence

The existing effect code is heavily balance-movement oriented, so a reviewer may
argue that SAC authorization events were intentionally left out until new effect
types are designed. The omission is still meaningful for downstream users,
because classic asset authorization changes already appear in `history_effects`
and the SAC analogue silently breaks that parity.
