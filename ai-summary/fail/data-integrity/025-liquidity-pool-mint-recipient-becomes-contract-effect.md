# H002: Liquidity-pool SAC mint recipients become contract effects

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a Stellar Asset Contract mint credits a liquidity pool, the exported effect row should identify the minted `L...` pool recipient. The row should not attribute that credit to the invoking operation's source account or relabel the pool as a contract.

## Mechanism

The mint branch in `addInvokeHostFunctionEffects()` uses `strkey.IsValidEd25519PublicKey(mintEvent.To)` as its only positive account test. A legitimate liquidity-pool recipient therefore misses the account branch and flows into the fallback that sets `details["contract"] = mintEvent.To` and emits `addMuxed(source, EffectContractCredited, ...)`. The resulting effect row silently claims that the source account was credited, while the real liquidity-pool recipient survives only as a mislabeled `contract` detail.

## Trigger

Export effects for any valid SAC mint whose recipient is a liquidity pool, such as the upstream liquidity-pool deposit fixture that emits `mintEvent(lpIdToStrkey(lpBtcEthId), ...)`. The corresponding effect row will use the operation source account as `address` and `contract_credited` as the type instead of preserving the credited `L...` pool.

## Target Code

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1380-1394` — mint recipient classification accepts only `G...` and otherwise emits a contract credit on the source account
- `internal/transform/operation.go:parseAssetBalanceChangesFromContractEvents/createSACBalanceChangeEntry:1926-1928,1984-1997` — sibling operation export preserves the minted recipient string without `G...`-only filtering
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/scval.go:34-39` — SDK stringification returns `L...` for liquidity-pool `ScAddress`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/processors/token_transfer/token_transfer_processor_test.go:1412-1421` — realistic SAC mint fixture credits a liquidity-pool address

## Evidence

The SDK and upstream token-transfer fixtures already model liquidity pools as mint recipients, so this is not a theoretical address type. Local `asset_balance_changes` export keeps the raw SAC recipient string, proving the ETL can already surface the actual `L...` holder on a sibling surface. The corruption is localized to the effects path's `G...`-only branch test plus the `addMuxed(source, ...)` fallback.

## Anti-Evidence

Maintainers may argue that the existing effect taxonomy has no dedicated liquidity-pool credit type and therefore routes all non-account holders through the contract bucket. But that does not justify exporting the source account as the credited entity. Even if a new type is needed, the current row still encodes the wrong participant.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of fail/data-integrity/017 (contract_debited/contract_credited source-account attribution)
**Failed At**: reviewer

### Trace Summary

Traced the mint branch in `addInvokeHostFunctionEffects()` at `internal/transform/effects.go:1380-1394` and compared it line-by-line against the upstream `stellar/go` standalone effects processor at `processors/effects/effects.go:1507-1521`. The two implementations are character-for-character identical: both use `strkey.IsValidEd25519PublicKey(mintEvent.To)` as the sole account test, and both route non-G... recipients (including L... liquidity pool addresses) through the `details["contract"] = mintEvent.To` / `addMuxed(source, EffectContractCredited, details)` fallback. This is the same upstream effect schema pattern already investigated and rejected in fail 017.

### Code Paths Examined

- `internal/transform/effects.go:1380-1394` — Local mint branch: identical `G...-only` test followed by contract fallback with `addMuxed(source, ...)`
- `stellar/go processors/effects/effects.go:1507-1521` — Upstream standalone processor: character-identical mint branch with same `IsValidEd25519PublicKey` gate and same `addMuxed(source, EffectContractCredited, ...)` fallback
- `internal/transform/effects.go:191-198` — `addMuxed` resolves the MuxedAccount to an AccountId and calls `e.add(accID.Address(), ...)`, confirming the source account becomes the `address` field

### Why It Failed

This hypothesis is a specific instance of the general pattern already rejected in fail 017: the `address=source` / `details.contract=actual_entity` split for `contract_credited`/`contract_debited` effects is the **established upstream effect schema**, not a local bug. The upstream `stellar/go` effects processor uses the identical code. Meta-pattern #5 in the fail summary explicitly warns: "Effects intentionally mirror `stellar/go` Horizon semantics. Before reporting an effect field mismatch (e.g., `address` vs `details.contract`), verify that the same behavior is present in the upstream processor."

The L... liquidity-pool angle does not create a materially distinct issue — the same code path handles all non-G... addresses (C... contracts, L... pools, M... muxed accounts) identically in both the local code and upstream. There is no special liquidity-pool branch in either implementation, and this is by design in the upstream effect taxonomy.

### Lesson Learned

When a hypothesis narrows a previously rejected pattern to a specific address type (L... instead of C...), verify whether the code path actually distinguishes between those types. If the same `else` branch handles all non-G... addresses uniformly in both local and upstream code, the narrowed hypothesis is a duplicate regardless of the address type cited.
