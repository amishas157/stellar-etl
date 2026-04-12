# H001: Trustline export omits live `liquidity_pool_use_count`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a trustline carries `TrustLineEntryExtensionV2.LiquidityPoolUseCount > 0`, the exported trustline row should preserve that counter in both JSON and Parquet output. Trustlines that are otherwise identical in balance, liabilities, flags, and pool ID but differ in liquidity-pool usage count should remain distinguishable in the exported state snapshot.

## Mechanism

Current XDR defines `TrustLineEntryExtensionV2.liquidityPoolUseCount` as a first-class trustline field, but `TransformTrustline()` never reads the extension chain and neither `TrustlineOutput` nor `TrustlineOutputParquet` has a destination column for it. Any ledger where the trustline V2 counter changes therefore exports a plausible-but-incomplete row that silently drops live trustline state.

## Trigger

Export trustline ledger-entry changes for a trustline whose XDR extension chain reaches `TrustLineEntryExtensionV2` and whose `LiquidityPoolUseCount` is non-zero. Compare the raw trustline XDR with the emitted JSON/Parquet row: the XDR contains the counter, but the export surface has no field for it.

## Target Code

- `internal/transform/trustline.go:17-90` — `TransformTrustline()` never reads `trustEntry.Ext...V2.LiquidityPoolUseCount`
- `internal/transform/schema.go:238-258` — `TrustlineOutput` has no `liquidity_pool_use_count` field
- `internal/transform/schema_parquet.go:171-191` — `TrustlineOutputParquet` also has no destination field
- `go-stellar-sdk/xdr/xdr_generated.go:4589-4605` — XDR defines `TrustLineEntryExtensionV2.LiquidityPoolUseCount`

## Evidence

The XDR comment and generated struct show `TrustLineEntryExtensionV2` contains `int32 liquidityPoolUseCount` as part of the trustline entry itself. The transform and both trustline schemas only export balances, liabilities, flags, and pool identifiers; there is no code path that can preserve the V2 counter.

## Anti-Evidence

Only trustlines that actually use the V2 extension are affected, so many trustline rows will never exercise this path. Some downstream consumers may be able to infer related pool activity from other tables, but the explicit per-trustline usage count is still silently lost from the trustline export itself.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full `TransformTrustline()` path in `internal/transform/trustline.go:17-90` and confirmed the function never accesses the V2 extension chain. Verified `TrustlineOutput` (schema.go:238-258) and `TrustlineOutputParquet` (schema_parquet.go:171-191) have no destination field for `LiquidityPoolUseCount`. However, investigation of the upstream `stellar/go` SDK revealed that Horizon — the canonical Stellar API — also does not export this field anywhere. The field is not referenced outside of auto-generated XDR code in the entire `stellar/go` codebase (no usage in services, processors, protocols, or ingest packages).

### Code Paths Examined

- `internal/transform/trustline.go:17-90` — `TransformTrustline()` extracts balance, liabilities, flags, pool ID, sponsor, but never reads the extension chain for V2 fields. Confirmed omission.
- `internal/transform/schema.go:238-258` — `TrustlineOutput` struct has no `LiquidityPoolUseCount` field. Confirmed.
- `internal/transform/schema_parquet.go:171-191` — `TrustlineOutputParquet` also has no destination field. Confirmed.
- `go-stellar-sdk/xdr/xdr_generated.go:4602-4605` — `TrustLineEntryExtensionV2` contains `LiquidityPoolUseCount Int32`. Field exists in XDR.
- `stellar/go@v0.0.0-20250915171319-4914d3d0af61` (full SDK search) — `LiquidityPoolUseCount` appears ONLY in auto-generated XDR files (`xdr/xdr_generated.go`, `gxdr/xdr_generated.go`). Zero references in Horizon services, ingest processors, protocol types, or API handlers.

### Why It Failed

This is **working-as-designed behavior**, not a bug. The `LiquidityPoolUseCount` field is a protocol-level reference counter that tracks how many liquidity pool share trustlines reference a given asset trustline. It exists to enforce protocol invariants (preventing deletion of a trustline still needed by a pool) — it is not business-meaningful analytics data.

The strongest evidence that this omission is intentional: **Horizon itself — the canonical Stellar blockchain API maintained by SDF — also does not export this field.** A search of the entire `stellar/go` SDK reveals zero non-XDR references to `LiquidityPoolUseCount`. No processor reads it, no API serializes it, no database column stores it. The stellar-etl's BigQuery schema is designed to align with the Horizon export surface, and this field is consistently omitted across the entire Stellar data ecosystem.

The hypothesis mischaracterizes an internal protocol bookkeeping counter as "live trustline state" that should appear in analytics exports.

### Lesson Learned

The presence of a field in XDR does not mean it should appear in data exports. Protocol-internal bookkeeping fields (reference counters, invariant enforcement) may be intentionally omitted from all consumer-facing data surfaces. Before claiming an XDR field is "silently dropped," verify whether the canonical upstream system (Horizon) exports it — if Horizon omits it too, the omission is likely by design.
