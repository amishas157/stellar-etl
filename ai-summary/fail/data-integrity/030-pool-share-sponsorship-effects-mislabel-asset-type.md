# H003: Pool-share sponsorship effects mislabel the trustline asset as `liquidity_pool`

**Date**: 2026-04-15
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`trustline_sponsorship_created`, `trustline_sponsorship_updated`, and `trustline_sponsorship_removed` effects for liquidity-pool-share trustlines should classify the asset consistently with other pool-share trustline effects. The details should use `asset_type = liquidity_pool_shares` and preserve the same canonical identifiers (`liquidity_pool_id`, `liquidity_pool_id_strkey`) that `trustline_created` already exports for the same underlying trustline type.

## Mechanism

The generic trustline-change effect formatter special-cases pool-share trustlines by hard-coding `asset_type = "liquidity_pool"` and only `liquidity_pool_id`. That creates a second, incompatible representation for the same trustline class: elsewhere in the same package, pool-share trustlines are represented as `asset_type = liquidity_pool_shares` with both hex and StrKey pool identifiers. Consumers grouping or joining by `asset_type` therefore split one semantic entity across two incompatible encodings.

## Trigger

Export effects for any begin-sponsoring / end-sponsoring / revoke-sponsorship flow that changes sponsorship on a liquidity-pool-share trustline. The sponsorship effect rows will advertise `asset_type = liquidity_pool` and omit `liquidity_pool_id_strkey`, while a `trustline_created` effect for the same pool-share trustline uses `asset_type = liquidity_pool_shares` and includes the StrKey.

## Target Code

- `internal/transform/effects.go:338-346` — generic trustline-change formatter rewrites pool-share trustlines to `asset_type = liquidity_pool`
- `internal/transform/effects.go:682-692` — pool-share `change_trust` effects preserve the canonical pool-share representation
- `internal/transform/effects_test.go:2498-2534` — tests currently pin `asset_type = liquidity_pool` for sponsorship effects
- `internal/transform/effects_test.go:2692-2699` — tests pin `asset_type = liquidity_pool_shares` plus `liquidity_pool_id_strkey` for `trustline_created`

## Evidence

The trustline-change formatter explicitly sets `details["asset_type"] = "liquidity_pool"` and `details["liquidity_pool_id"] = ...` for pool-share trustlines. The package's own tests confirm that sponsorship effects use this shape, while a separate test for `trustline_created` on the same pool-share asset expects `asset_type = liquidity_pool_shares` and a populated `liquidity_pool_id_strkey`. That demonstrates an internally inconsistent encoding for the same asset class inside `history_effects`.

## Anti-Evidence

This exact representation also appears in upstream Horizon effects code, so the bug may be a long-lived inherited convention rather than a local regression. Consumers that key only on `liquidity_pool_id` can still correlate the rows, but any pipeline that relies on `asset_type` normalization or the StrKey join key will silently split sponsorship effects away from the rest of the pool-share trustline history.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

The hypothesis identifies a real inconsistency between `asset_type = "liquidity_pool"` in sponsorship effects (effects.go:341-343) and `asset_type = "liquidity_pool_shares"` in change_trust effects (via `addLiquidityPoolAssetDetails` at operation.go:390). However, tracing the upstream Horizon code at `stellar/go@v0.0.0-20250915171319-4914d3d0af61/services/horizon/internal/ingest/processors/effects_processor.go:381-383` reveals the exact same pattern: sponsorship effects use `"liquidity_pool"` while `addLiquidityPoolAssetDetails` uses `"liquidity_pool_shares"`. The stellar-etl code is a faithful mirror of upstream Horizon semantics, making this working-as-designed behavior.

### Code Paths Examined

- `internal/transform/effects.go:290-370` (`addLedgerEntrySponsorshipEffects`) — sponsorship handler sets `asset_type = "liquidity_pool"` and `liquidity_pool_id` for pool-share trustlines at lines 341-343
- `internal/transform/effects.go:682-692` — change_trust handler delegates to `addLiquidityPoolAssetDetails` which sets `asset_type = "liquidity_pool_shares"`, `liquidity_pool_id`, and `liquidity_pool_id_strkey`
- `internal/transform/operation.go:389-406` (`addLiquidityPoolAssetDetails`) — local helper includes StrKey encoding not present in upstream
- `upstream effects_processor.go:381-383` — upstream Horizon uses identical `"liquidity_pool"` for sponsorship effects on pool-share trustlines
- `upstream operations_processor.go:837-849` — upstream `addLiquidityPoolAssetDetails` uses `"liquidity_pool_shares"` (same split)
- `internal/transform/effects_test.go:2498-2534` — tests pin `asset_type = "liquidity_pool"` for sponsorship effects, confirming intentional behavior

### Why It Failed

This describes **working-as-designed behavior** that intentionally mirrors upstream Horizon. Per the established meta-pattern (#5 in fail/data-integrity/summary.md): "Effects intentionally mirror `stellar/go` Horizon semantics. Before reporting an effect field mismatch, verify that the same behavior is present in the upstream `processors/effects/effects.go` and its tests." The upstream Horizon code at `effects_processor.go:381-383` uses the identical `"liquidity_pool"` asset type for sponsorship effects on pool-share trustlines. The `"liquidity_pool_shares"` string comes from a different code path (`addLiquidityPoolAssetDetails`) that has access to `LiquidityPoolParameters` and is used for change_trust effects — a different effect category. The inconsistency is a long-standing upstream convention, not a stellar-etl regression.

### Lesson Learned

The `asset_type` naming difference between sponsorship effects (`"liquidity_pool"`) and change_trust effects (`"liquidity_pool_shares"`) is an inherited upstream Horizon convention. The two paths use different data sources: sponsorship effects read from `TrustLineEntry.Asset` (a `TrustLineAsset` union that carries only the pool ID hash), while change_trust effects read from `ChangeTrustOp.Line` (which carries `LiquidityPoolParameters` with the full asset pair). The different naming reflects these different data access patterns and is consistent across both upstream Horizon and stellar-etl.
