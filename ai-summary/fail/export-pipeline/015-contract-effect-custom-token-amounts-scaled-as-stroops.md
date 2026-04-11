# H001: Contract-token effects scale custom SEP-41 amounts as 7-decimal stroops

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_effects.details.amount` should preserve the amount carried by a contract token event. For a non-SAC SEP-41 transfer/mint/burn/clawback whose emitted i128 amount is `1000`, the effect row should export `details.amount = "1000"` unless the code has token-specific decimal metadata proving a different scale.

## Mechanism

The contract-effect path formats every token amount with `amount.String128(...)`, and that helper explicitly assumes fixed 7-decimal Stellar Classic precision. Custom SEP-41 tokens are not universally 7-decimal assets, so the effects export silently divides their raw i128 amounts by `10^7`, producing plausible-looking but financially wrong effect rows.

## Trigger

Export any Soroban `invoke_host_function` transaction whose contract events include a non-SAC SEP-41 `transfer`, `mint`, `burn`, or `clawback`. A raw contract amount like `1000` will be emitted in `history_effects.details.amount` as `0.0001000` instead of `1000`.

## Target Code

- `internal/transform/effects.go:1342-1417` — contract event effects write `details["amount"] = amount.String128(...)` for transfer/mint/clawback/burn
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/amount/main.go:138-152` — `String128` "boldly assum[es] 7-decimal precision"

## Evidence

The effect exporter does not branch on token type before formatting the amount: all four contract-token effect arms call the same 7-decimal helper. The helper's own comment says that assumption is only the "correct default for Stellar Classic amounts," which does not hold for arbitrary SEP-41 contract tokens.

## Anti-Evidence

For native/SAC-backed token events, 7-decimal formatting is the intended scale, so those rows would still look correct. The bug matters specifically when the contract event represents a custom token whose amount is already in raw token units.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not a duplicate of prior fail/success items (success/009 covers a different code path in token_transfer.go)
**Failed At**: reviewer

### Trace Summary

The hypothesis claims that `effects.go` applies `amount.String128()` (7-decimal stroop scaling) to custom SEP-41 token events. However, the effects code path has an explicit guard that prevents this. At `effects.go:1329`, `contractevents.NewStellarAssetContractEvent()` performs rigorous SAC validation — checking contract ID, topic format, canonical asset string, and contract ID integrity against the network passphrase. Non-SAC custom SEP-41 events fail this validation and are skipped via `continue` at line 1331. The `amount.String128()` calls at lines 1348/1383/1401/1417 are only ever reached for genuine SAC events, for which 7-decimal precision is correct.

### Code Paths Examined

- `internal/transform/effects.go:1322-1430` — `addInvokeHostFunctionEffects()`: iterates events, calls `NewStellarAssetContractEvent()` as a filter, skips non-SAC events
- `internal/transform/effects.go:1329-1331` — THE GUARD: `NewStellarAssetContractEvent()` returns error for non-SAC events → `continue`
- `go-stellar-sdk/support/contractevents/event.go:61-130` — `NewStellarAssetContractEvent()`: validates event type, contract ID, topics, canonical asset string, and contract ID integrity; returns `ErrNotStellarAssetContract` for any non-SAC event
- `go-stellar-sdk/amount/main.go:134-148` — `String128()`: divides i128 by 10^7, confirmed to assume 7-decimal precision

### Why It Failed

The hypothesis assumes custom SEP-41 token events reach the `amount.String128()` formatting calls, but they do not. `NewStellarAssetContractEvent()` acts as a strict SAC-only filter — it validates that the event's contract ID matches the expected SAC contract ID derived from the asset and network passphrase. Any event from a custom (non-SAC) contract fails this check and is skipped. Therefore, `amount.String128()` is only ever called on SAC events where 7-decimal stroop precision is correct by definition.

Note: The *related* bug in `token_transfer.go` (success/009) is real because that code path uses a different SDK entry point (`token_transfer.TokenTransferEvent`) that does process custom SEP-41 tokens and applies stroop scaling unconditionally. The effects path is protected by its SAC-specific filter.

### Lesson Learned

When a guard function like `NewStellarAssetContractEvent()` returns an error that causes a `continue`, all code below that guard in the loop body is unreachable for the filtered-out cases. Always trace the full loop from its entry point, not just the formatting call in isolation. The effects path and token_transfer path use different SDK entry points with different filtering behavior — they cannot be assumed to share the same bug.
