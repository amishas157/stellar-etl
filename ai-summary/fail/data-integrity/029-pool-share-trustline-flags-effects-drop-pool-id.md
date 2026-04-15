# H002: Pool-share `trustline_flags_updated` effects drop the target pool identifier

**Date**: 2026-04-15
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `set_trust_line_flags` updates flags on a liquidity-pool-share trustline, the resulting `history_effects` row (`trustline_flags_updated`) should identify the exact pool-share trustline being modified. As with other pool-share trustline effects, the effect details should include `asset_type = liquidity_pool_shares`, `liquidity_pool_id`, and `liquidity_pool_id_strkey`.

## Mechanism

`addSetTrustLineFlagsEffects()` delegates to `addTrustLineFlagsEffect()`, which formats the target asset with the generic `addAssetDetails()` helper. That helper only emits `asset_type`, `asset_code`, and `asset_issuer`; for pool shares, `Extract()` provides only the type, so the effect keeps `asset_type = liquidity_pool_shares` but drops both pool identifiers entirely. The same package's pool-share `change_trust` effect path already uses `addLiquidityPoolAssetDetails()`, so flag-update effects lose identity that sibling trustline effects preserve.

## Trigger

Export effects for any `set_trust_line_flags` operation that targets a liquidity-pool-share trustline, such as clearing authorization on LP shares. The emitted `trustline_flags_updated` effect will contain `trustor` and flag booleans, but no `liquidity_pool_id` / `liquidity_pool_id_strkey`, making the affected pool-share trustline impossible to identify from the effect row alone.

## Target Code

- `internal/transform/effects.go:addTrustLineFlagsEffect:1101-1124` — pool-share flag effects go through generic asset formatting
- `internal/transform/operation.go:addAssetDetails:2001-2020` — generic helper drops pool-specific identifiers
- `internal/transform/effects.go:682-692` — pool-share `change_trust` effects already use `addLiquidityPoolAssetDetails()`

## Evidence

`addTrustLineFlagsEffect()` unconditionally calls `addAssetDetails(details, asset, "")` before adding the authorization flags. `addAssetDetails()` does not know about pool shares and therefore cannot emit `liquidity_pool_id` or `liquidity_pool_id_strkey`. By contrast, the pool-share branch in the `change_trust` effect code explicitly calls `addLiquidityPoolAssetDetails()` and preserves both identifiers, showing the omission is not an unavoidable schema limitation.

## Anti-Evidence

Upstream Horizon's effects processor uses the same helper chain, so this may reflect inherited behavior rather than a stellar-etl-only fork bug. There is also no existing test that exercises pool-share `trustline_flags_updated`, which means the current output shape has not been pinned as explicitly as the sponsorship paths below.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantially equivalent to fail/data-integrity/028-set-trust-line-flags-pool-share-details-collapse.md
**Failed At**: reviewer

### Trace Summary

The hypothesis claims that `addTrustLineFlagsEffect()` loses pool-share identity when processing `set_trust_line_flags` operations targeting liquidity-pool-share trustlines. However, `SetTrustLineFlagsOp.Asset` is of type `xdr.Asset`, which is a union with only three arms: `ASSET_TYPE_NATIVE`, `ASSET_TYPE_CREDIT_ALPHANUM4`, and `ASSET_TYPE_CREDIT_ALPHANUM12`. The `ASSET_TYPE_POOL_SHARE` arm only exists in the `TrustLineAsset` and `ChangeTrustAsset` unions. Therefore, pool shares can never reach `addTrustLineFlagsEffect()` — the trigger condition is impossible at the XDR protocol level.

### Code Paths Examined

- `internal/transform/effects.go:1094-1098` (`addSetTrustLineFlagsEffects`) — passes `op.Asset` (type `xdr.Asset`) to `addTrustLineFlagsEffect`
- `internal/transform/effects.go:1101-1125` (`addTrustLineFlagsEffect`) — takes `asset xdr.Asset` parameter, calls `addAssetDetails(details, asset, "")`
- `internal/transform/effects.go:700-731` (`addAllowTrustEffects`) — other caller, also passes `xdr.Asset` from `AllowTrustOp.Asset.ToAsset()`
- `internal/transform/operation.go:2001-2021` (`addAssetDetails`) — processes `xdr.Asset` via `Extract()`, emits `asset_type`, `asset_code`, `asset_issuer`
- `internal/transform/effects.go:682-692` — `change_trust` path correctly special-cases pool shares because `ChangeTrustOp.Line` is `xdr.ChangeTrustAsset` (which includes the pool-share arm)

### Why It Failed

This hypothesis is substantially equivalent to the already-investigated 028-set-trust-line-flags-pool-share-details-collapse.md, and fails for the identical reason: `SetTrustLineFlagsOp.Asset` is `xdr.Asset`, which cannot represent pool shares. The XDR type system structurally prevents pool shares from reaching `addTrustLineFlagsEffect()`. The `change_trust` path correctly special-cases pool shares because `ChangeTrustOp.Line` uses `xdr.ChangeTrustAsset`, a different union type that includes the pool-share arm. The difference in special-case handling between `change_trust` and `set_trust_line_flags` is not an omission — it reflects the different XDR type constraints of the two operations.

### Lesson Learned

The same XDR type constraint (`xdr.Asset` vs `xdr.TrustLineAsset`/`xdr.ChangeTrustAsset`) applies to both the operation details path and the effects path for `set_trust_line_flags`. Hypotheses about pool-share identity loss should verify the XDR union type at the entry point, not just the helper function's handling. This is the same lesson as 028 applied to the effects code path.
