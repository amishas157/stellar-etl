# H064: Pool-share asset labels diverge across tables

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If Stellar ETL uses a single canonical asset-type label for liquidity-pool shares, then trustline, operation, and effect exports should all emit the same string for the same pool-share asset. A pool-share trustline and a pool-share operation detail should not disagree on the human-readable asset-type token if downstream queries are expected to join them directly on that label alone.

## Mechanism

`TransformTrustline()` hard-codes pool-share trustlines as `asset_type = "pool_share"`, while operation detail helpers emit `asset_type = "liquidity_pool_shares"` for liquidity-pool assets. That initially looked like a copy/paste mapping bug that would make cross-table asset-type filters silently miss rows.

## Trigger

Export a ledger that contains both a pool-share trustline row and a liquidity-pool-related operation/effect row, then compare the exported asset-type labels across those tables.

## Target Code

- `internal/transform/trustline.go:TransformTrustline:43-57` — pool-share trustlines force `asset_type = "pool_share"`
- `internal/transform/operation.go:addLiquidityPoolAssetDetails:389-406` — operation details emit `asset_type = "liquidity_pool_shares"`

## Evidence

The two transform paths really do emit different strings for closely related liquidity-pool assets. The mismatch is visible directly in the code, and at first glance it looks like one side is exporting the wrong semantic label.

## Anti-Evidence

The operation-side label matches the upstream Horizon ingest processor's `addLiquidityPoolAssetDetails()` helper, which also emits `liquidity_pool_shares` for operation detail payloads. Separately, local trustline expectations already treat `pool_share` as the established trustline-table label rather than a mistaken spelling.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is a table-contract split, not a live wrong-value bug in a single export path. Operation/effect detail payloads intentionally mirror upstream Horizon naming, while trustline rows keep their own long-standing `pool_share` label.

### Lesson Learned

Do not assume that similarly named asset-type fields across different exported tables share one global string contract. For liquidity-pool surfaces in particular, trustline rows and operation/effect detail payloads can intentionally use different legacy labels.
