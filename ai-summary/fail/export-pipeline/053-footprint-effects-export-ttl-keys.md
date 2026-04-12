# H053: Footprint effects export TTL keys instead of the referenced footprint keys

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `extend_footprint_ttl` and `restore_footprint` effect rows are meant to identify the user-visible ledger entries named in the Soroban footprint, the `details.entries` list should preserve those underlying contract-data / contract-code ledger keys. A downstream consumer should be able to decode each entry and recover the same footprint members the operation targeted, rather than an internal side-table key.

## Mechanism

`addExtendFootprintTtlEffect()` and `addRestoreFootprintExpirationEffect()` iterate the operation's post-changes, keep only `LedgerEntryTypeTtl`, and reconstruct `LedgerKeyTtl` values from `TtlEntry.KeyHash`. That looked like a plausible export bug because the emitted `entries` list then describes mutated TTL rows instead of the original footprint keys users conceptually extended or restored.

## Trigger

1. Export effects for a ledger containing `ExtendFootprintTtlOp` or `RestoreFootprintOp`.
2. Decode `details.entries`.
3. Compare those decoded keys with the operation footprint and observe that the export lists `LedgerKeyTtl` rows keyed by hash, not the original contract-data / contract-code keys.

## Target Code

- `internal/transform/effects.go:addExtendFootprintTtlEffect:1435-1474` — rebuilds `entries` from `TtlEntry.KeyHash` via `LedgerKey.SetTtl`
- `internal/transform/effects.go:addRestoreFootprintExpirationEffect:1477-1511` — same pattern for restore
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/effects/effects.go:1565-1642` — upstream Horizon processor mirrors the same TTL-key encoding

## Evidence

The local exporter never looks at the operation footprint here; it only inspects post-operation TTL ledger-entry changes and serializes `LedgerKeyTtl` values rebuilt from `v.KeyHash`. Upstream `stellar/go`'s effects processor uses the same logic, and both local and upstream tests assert `details["entries"]` against the same TTL-key base64 strings.

## Anti-Evidence

The effect rows are intentionally derived from the ledger entries actually mutated by these operations, and those mutated rows are TTL entries. The upstream processor and test suite treat TTL-key encoding as the canonical contract for this effect family, not as an accidental substitute for the original footprint.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is upstream-defined behavior, not a local export bug. `extend_footprint_ttl` and `restore_footprint` effects intentionally report the TTL ledger entries whose expirations were mutated, and both stellar-etl and the upstream Horizon processor/tests encode those entries as `LedgerKeyTtl` base64 strings.

### Lesson Learned

For Soroban archival effects, "affected entries" can mean the internal TTL side-table rows that changed, not the original footprint members a user might expect. Before filing a footprint-key mismatch, check the upstream effects processor and tests to see whether the ETL is mirroring a deliberate Horizon contract.
