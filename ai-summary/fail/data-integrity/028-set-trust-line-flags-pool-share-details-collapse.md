# H001: Pool-share `set_trust_line_flags` details lose liquidity-pool identity

**Date**: 2026-04-15
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a `set_trust_line_flags` operation targets a liquidity-pool-share trustline, the exported `history_operations.details` payload should identify the exact pool-share asset, the same way sibling trustline code paths already do. The row should carry `asset_type = liquidity_pool_shares` plus the concrete `liquidity_pool_id` and `liquidity_pool_id_strkey`, rather than collapsing every pool-share trustline into the same generic asset shape.

## Mechanism

`extractOperationDetails()` routes `OperationTypeSetTrustLineFlags` through `addAssetDetailsToOperationDetails()`, a helper that only understands native and credit assets. For pool shares, `xdr.Asset.Extract()` yields only the type string, so the helper emits empty `asset_code` / `asset_issuer`, omits both pool identifiers, and hashes the empty tuple into one shared `asset_id` for every pool-share flag update. Sibling paths in the same file (`change_trust`, trustline ledger-key formatting) already special-case pool shares, so this branch is a lossy outlier.

## Trigger

Export operations for any ledger containing a `set_trust_line_flags` operation whose `asset` is `AssetTypeAssetTypePoolShare`. Compare the resulting `history_operations.details` row against the same pool's `change_trust` / revoke-sponsorship details: the flag-update row will lack `liquidity_pool_id` / `liquidity_pool_id_strkey` and will expose a generic pool-share `asset_id` shared across unrelated pools.

## Target Code

- `internal/transform/operation.go:addAssetDetailsToOperationDetails:367-386` — generic asset formatter hashes empty pool-share code/issuer tuples
- `internal/transform/operation.go:addLiquidityPoolAssetDetails:389-406` — sibling helper that preserves pool-share identity
- `internal/transform/operation.go:extractOperationDetails:943-957` — `set_trust_line_flags` uses the generic helper
- `internal/transform/operation.go:extractOperationDetails:806-818` — `change_trust` already special-cases pool-share assets

## Evidence

The `set_trust_line_flags` branch calls `addAssetDetailsToOperationDetails(details, op.Asset, "")` directly. That helper writes `asset_type`, `asset_code`, `asset_issuer`, and `asset_id`, but it never emits `liquidity_pool_id`; for non-native assets it always computes `asset_id = FarmHashAsset(code, issuer, assetType)`, and `code` / `issuer` stay empty for `AssetTypeAssetTypePoolShare`. In the same file, pool-share `change_trust` uses `addLiquidityPoolAssetDetails()` instead, proving the exporter already has a canonical representation for the same asset class.

## Anti-Evidence

Current Horizon ingest code mirrors the same generic `set_trust_line_flags` formatting, so this may be a long-standing inherited quirk rather than a local one-off regression. Downstream consumers can still recover the true pool identifier by decoding the raw transaction XDR, but the purpose of `details` is to expose that information directly, and this branch fails to do so.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

The hypothesis claims that `SetTrustLineFlagsOp.Asset` can be `AssetTypeAssetTypePoolShare`, causing pool-share trustline flag updates to lose their liquidity pool identity. However, the XDR specification for `SetTrustLineFlagsOp` uses the `Asset` union type (not `TrustLineAsset`), and the `Asset` union only has three arms: `ASSET_TYPE_NATIVE`, `ASSET_TYPE_CREDIT_ALPHANUM4`, and `ASSET_TYPE_CREDIT_ALPHANUM12`. The `ASSET_TYPE_POOL_SHARE` arm exists only in the `TrustLineAsset` union (used by `ChangeTrustOp`). Therefore, `SetTrustLineFlagsOp.Asset` can never be a pool share — the trigger condition is impossible at the XDR protocol level.

### Code Paths Examined

- `xdr.SetTrustLineFlagsOp` struct definition — `Asset` field is type `xdr.Asset`, NOT `xdr.TrustLineAsset`
- `xdr.Asset` union definition — arms: `ASSET_TYPE_NATIVE`, `ASSET_TYPE_CREDIT_ALPHANUM4`, `ASSET_TYPE_CREDIT_ALPHANUM12` only
- `xdr.TrustLineAsset` union definition — adds `ASSET_TYPE_POOL_SHARE` arm with `PoolID liquidityPoolID`
- `internal/transform/operation.go:943-955` — `OperationTypeSetTrustLineFlags` case calls `addAssetDetailsToOperationDetails(details, op.Asset, "")` which is correct for the `xdr.Asset` type that can only be native or credit
- `internal/transform/operation.go:806-818` — `ChangeTrustOp` correctly special-cases pool shares because `ChangeTrustOp.Line` is `xdr.ChangeTrustAsset` which DOES include pool shares

### Why It Failed

The hypothesis is based on an incorrect assumption about the XDR type system. `SetTrustLineFlagsOp.Asset` is `xdr.Asset`, which cannot represent pool shares. Pool shares can only appear in `TrustLineAsset` and `ChangeTrustAsset` unions. The `change_trust` path special-cases pool shares because `ChangeTrustOp.Line` IS a type that can hold pool shares. The `set_trust_line_flags` path does not need this special case because its `Asset` field structurally cannot be a pool share. The code is correct as written.

### Lesson Learned

Before hypothesizing that a code path is missing a pool-share special case, verify the XDR union type of the field being processed. The Stellar XDR distinguishes between `Asset` (native + credit only), `TrustLineAsset` (adds pool shares), and `ChangeTrustAsset` (adds pool shares). Not all asset-like fields can carry pool shares — only those using the extended union types.
