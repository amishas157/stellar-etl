# H003: Claimable-balance SAC clawback senders become contract effects

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a Stellar Asset Contract clawback debits a claimable balance, the exported effect row should identify the actual `B...` claimable-balance holder that lost funds. The row should not be rebound to the operation source account, and the claimable balance should not be mislabeled as a contract.

## Mechanism

The clawback branch checks `strkey.IsValidEd25519PublicKey(cbEvent.From)` and treats every non-`G...` sender as though it were a contract. Claimable-balance strkeys begin with `B`, so legitimate clawback senders miss the account path and fall into the fallback that sets `details["contract"] = cbEvent.From` and emits `addMuxed(source, EffectContractDebited, ...)`. The exported row therefore says the source account was debited by a contract-style effect even though the debited holder was the claimable balance.

## Trigger

Export effects for any valid SAC clawback whose source holder is a claimable balance, such as the upstream claimable-balance clawback fixtures that emit `clawbackEvent(cbIdToStrkey(...), ...)`. The resulting effect row will show the operation source account in `address` while the real `B...` holder appears only under `details.contract`.

## Target Code

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1398-1411` — clawback sender classification accepts only `G...` and otherwise emits a contract debit on the source account
- `internal/transform/operation.go:parseAssetBalanceChangesFromContractEvents/createSACBalanceChangeEntry:1929-1931,1984-1997` — sibling operation export preserves the clawed-back sender string in `from`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/scval.go:37-39` — `ScAddress.String()` encodes claimable-balance participants as `B...`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/processors/token_transfer/token_transfer_processor_test.go:1519-1543` — realistic token-transfer fixtures emit clawback events whose sender is a claimable balance

## Evidence

The current SDK exposes claimable balances as first-class `ScAddress` values and upstream fixtures already produce real SAC clawback events with `B...` senders. Local operation export preserves those senders verbatim in `asset_balance_changes.from`, so the ETL already has a working representation of the participant. Only the effects path collapses the `B...` sender into a fake contract debit on the operation source account.

## Anti-Evidence

The local effect taxonomy does not currently include a dedicated claimable-balance debit variant, and upstream Horizon mirrors the same broad `G...` vs non-`G...` split. That may make this look like inherited behavior. But the current output is still wrong because it attributes the debit to the wrong entity, not just to an imprecise category.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of fail/data-integrity/017 (contract_debited/contract_credited source-account attribution)
**Failed At**: reviewer

### Trace Summary

Traced the clawback branch in `addInvokeHostFunctionEffects()` at `internal/transform/effects.go:1398-1412` and compared it against the upstream `stellar/go` Horizon effects processor at `services/horizon/internal/ingest/processors/effects_processor.go:1581-1595`. The two implementations are functionally identical: both use `strkey.IsValidEd25519PublicKey(cbEvent.From)` as the sole account test, and both route non-G... senders (including B... claimable-balance addresses) through the `details["contract"] = evt.From` / `addMuxed(source, EffectContractDebited, details)` fallback. This is the same upstream effect schema pattern already investigated and rejected in fail 017, and subsequently re-rejected for L... liquidity-pool addresses in fail 025 (both transfer and mint variants).

### Code Paths Examined

- `internal/transform/effects.go:1398-1412` — Local SAC clawback branch: `IsValidEd25519PublicKey(cbEvent.From)` check with `EffectContractDebited` fallback via `addMuxed(source, ...)`
- `stellar/go effects_processor.go:1581-1595` — Upstream Horizon processor: character-identical clawback branch with same `IsValidEd25519PublicKey(evt.From)` gate and same `addMuxed(source, EffectContractDebited, details)` fallback
- `go-stellar-sdk xdr/scval.go:37-39` — `ScAddress.String()` does encode `ScAddressTypeClaimableBalance` as `B...` via `EncodeToStrkey()`, confirming such addresses can appear in SAC events
- `internal/transform/effects.go:191-198` — `addMuxed` resolves the MuxedAccount to an AccountId and calls `e.add(accID.Address(), ...)`, confirming the source account becomes the `address` field

### Why It Failed

This hypothesis is a specific instance of the general pattern already rejected in fail 017 and re-rejected twice in fail 025. The `address=source` / `details.contract=actual_entity` split for `contract_debited`/`contract_credited` effects is the **established upstream effect schema**, not a local bug. The upstream `stellar/go` Horizon effects processor uses identical code at lines 1581-1595. Meta-pattern #5 in the fail summary explicitly warns: "Effects intentionally mirror `stellar/go` Horizon semantics." The B... claimable-balance angle does not create a materially distinct issue — the same `else` branch handles all non-G... addresses (C... contracts, L... pools, B... claimable balances, M... muxed accounts) identically in both local and upstream code.

### Lesson Learned

The `address=source, details.contract=actual_entity` pattern for non-account SAC participants has now been rejected for C... contracts (fail 017), L... liquidity pools (fail 025 x2), and B... claimable balances (this investigation). All non-G... address types flow through the same undifferentiated `else` branch in both local and upstream code. No further variants of this hypothesis should be submitted for different address types — the rejection applies uniformly to any `ScAddress` type that is not `ScAddressTypeAccount`.
