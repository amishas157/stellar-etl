# H004: Claimable-balance SAC burn senders become contract effects

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a Stellar Asset Contract burn debits a claimable balance, the exported burn-side effect row should identify that `B...` claimable-balance sender. The row should not attribute the debit to the invoking source account or serialize the claimable balance as though it were a contract.

## Mechanism

The burn branch uses the same `strkey.IsValidEd25519PublicKey(burnEvent.From)` gate as the transfer/clawback logic. Because claimable-balance strkeys start with `B`, a legitimate burn sender misses the account branch and falls through to `details["contract"] = burnEvent.From` plus `addMuxed(source, EffectContractDebited, ...)`. That silently corrupts both the participant identity and the effect type: the export says the source account suffered a contract debit, while the real debited holder was the claimable balance.

## Trigger

Export effects for any valid SAC burn whose source holder is a claimable balance, such as the upstream claim-claimable-balance fixture that emits `burnEvent(cbIdToStrkey(...), ...)` when the claimed asset is burned instead of transferred. The resulting effect row will report the operation source account in `address` and stash the actual `B...` sender under `details.contract`.

## Target Code

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1414-1427` — burn sender classification accepts only `G...` and otherwise emits a contract debit on the source account
- `internal/transform/operation.go:parseAssetBalanceChangesFromContractEvents/createSACBalanceChangeEntry:1932-1934,1984-1997` — sibling operation export preserves the burn sender string in `from`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/scval.go:37-39` — `ScAddress.String()` encodes claimable-balance participants as `B...`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/processors/token_transfer/token_transfer_processor_test.go:1567-1578` — realistic token-transfer fixture emits a burn whose sender is a claimable balance

## Evidence

The SDK already serializes claimable-balance endpoints into `B...` strings, and upstream token-transfer fixtures prove that SAC burn events can legitimately use those addresses as senders. Local `asset_balance_changes` export preserves the same `B...` sender unchanged, so the ETL already has a lossless representation on a sibling surface. The effects exporter alone reclassifies that sender as a contract and substitutes the operation source account into `address`.

## Anti-Evidence

There is no dedicated claimable-balance burn effect type in the current enum set, and upstream Horizon presently follows the same broad fallback pattern. Even so, the present output is still materially wrong: it says the wrong entity was debited, which breaks downstream joins and per-entity balance/event attribution.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of fail/data-integrity/017 (contract_debited/contract_credited source-account attribution), fail/025 (L... variants), and fail/026 (B... clawback variant)
**Failed At**: reviewer

### Trace Summary

Traced the burn branch in `addInvokeHostFunctionEffects()` at `internal/transform/effects.go:1414-1428` and compared it against the upstream `stellar/go` standalone effects processor at `processors/effects/effects.go:1541-1555`. The two implementations are character-for-character identical: both use `strkey.IsValidEd25519PublicKey(burnEvent.From)` as the sole account test, and both route non-G... senders (including B... claimable-balance addresses) through the `details["contract"] = burnEvent.From` / `addMuxed(source, EffectContractDebited, details)` fallback. This is the same upstream effect schema pattern already investigated and rejected in fail 017, re-rejected for L... liquidity-pool addresses in fail 025 (transfer and mint), and re-rejected for B... claimable-balance clawback senders in fail 026.

### Code Paths Examined

- `internal/transform/effects.go:1414-1428` — Local SAC burn branch: `IsValidEd25519PublicKey(burnEvent.From)` check with `EffectContractDebited` fallback via `addMuxed(source, ...)`
- `stellar/go processors/effects/effects.go:1541-1555` — Upstream standalone processor: character-identical burn branch with same `IsValidEd25519PublicKey` gate and same `addMuxed(source, EffectContractDebited, details)` fallback
- `go-stellar-sdk xdr/scval.go:37-39` — `ScAddress.String()` encodes `ScAddressTypeClaimableBalance` as `B...` via `cbId.EncodeToStrkey()`, confirming such addresses can appear in SAC events
- `internal/transform/effects.go:191-198` — `addMuxed` resolves the MuxedAccount to an AccountId and calls `e.add(accID.Address(), ...)`, confirming the source account becomes the `address` field

### Why It Failed

This hypothesis is a specific instance of the general pattern already rejected in fail 017 and subsequently re-rejected in fail 025 (L... transfer and mint) and fail 026 (B... clawback). The `address=source` / `details.contract=actual_entity` split for `contract_debited`/`contract_credited` effects is the **established upstream effect schema**, not a local bug. The upstream `stellar/go` effects processor uses identical code at lines 1541-1555. Meta-pattern #5 in the fail summary explicitly warns: "Effects intentionally mirror `stellar/go` Horizon semantics." The B... claimable-balance burn angle does not create a materially distinct issue — the same `else` branch handles all non-G... addresses (C... contracts, L... pools, B... claimable balances, M... muxed accounts) identically in both local and upstream code. This is the fourth variant of the same hypothesis (after contracts, liquidity pools, and claimable-balance clawbacks).

### Lesson Learned

The `address=source, details.contract=actual_entity` pattern for non-account SAC participants has now been rejected for C... contracts (fail 017), L... liquidity pools (fail 025 x2), B... claimable-balance clawbacks (fail 026), and B... claimable-balance burns (this investigation). All four SAC event branches (transfer, mint, clawback, burn) use the identical `IsValidEd25519PublicKey` → `else` → contract fallback pattern, and all mirror the upstream Horizon processor exactly. No further variants should be submitted for any combination of event type × address type.
