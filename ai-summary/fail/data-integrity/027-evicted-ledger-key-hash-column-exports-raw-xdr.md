# H002: `evicted_ledger_keys_hash` exports raw XDR instead of ledger-key hashes

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_ledgers.evicted_ledger_keys_hash` should contain deterministic hashes of each evicted ledger key, matching the repository's `LedgerKeyToLedgerKeyHash()` helper and the semantics implied by the column name. A consumer should be able to compare an evicted key hash against other `ledger_key_hash` fields without re-parsing raw XDR.

## Mechanism

`transformLedgerKeys()` names its first output `ledgerKeysHash`, but fills it with `xdr.MarshalBase64(key)` rather than hashing the key bytes. Every ledger row with Soroban evictions therefore exports base64-encoded raw ledger keys like `AAAABQE...` under a `*_hash` column, which silently breaks joins and equality checks against the actual SHA-256 ledger-key hashes used elsewhere in the codebase.

## Trigger

1. Export ledgers for any Protocol 23+ range whose `LedgerCloseMeta` contains non-empty `EvictedKeys`.
2. Compare one exported `evicted_ledger_keys_hash` element to `utils.LedgerKeyToLedgerKeyHash()` of the same XDR `LedgerKey`.
3. Observe that the export contains raw base64 XDR, not the hex hash string.

## Target Code

- `internal/transform/ledger.go:73-76` — reads `EvictedKeys` into the exported ledger row
- `internal/transform/ledger.go:125-128` — writes `outputEvictedKeysHash` into `TotalByteSize...` row fields
- `internal/transform/ledger.go:213-224` — `transformLedgerKeys()` uses `xdr.MarshalBase64(key)` while naming the result `keyHash`
- `internal/utils/main.go:1043-1048` — `LedgerKeyToLedgerKeyHash()` shows the repository's actual hash semantics for ledger keys

## Evidence

The same repository already has a dedicated helper that marshals a ledger key to bytes, hashes those bytes, and hex-encodes the digest. `transformLedgerKeys()` instead base64-encodes the raw XDR and stores that value in `evicted_ledger_keys_hash`, which is why current fixtures show `AAAAB...`-style serialized payloads rather than digest-shaped hashes.

## Anti-Evidence

If downstream consumers have implicitly learned that this column contains raw key encodings despite its name, they may still be able to recover the true hash client-side. But the exported value is still not the hash the column advertises, and it diverges from the ETL's own sibling `ledger_key_hash` fields.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `transformLedgerKeys()` in `ledger.go:213-224` which uses `xdr.MarshalBase64(key)` for the evicted keys column. Then traced `TransformRestoredKey()` in `restored_key.go:26` which uses the identical `xdr.MarshalBase64(key)` approach for `RestoredKeyOutput.LedgerKeyHash`. Both evicted keys and restored keys consistently use base64-encoded XDR serialization, not SHA-256 hashes. The `ContractDataOutput` schema explicitly distinguishes the two formats with separate `ledger_key_hash` (SHA-256) and `ledger_key_hash_base_64` (base64 XDR) columns, confirming the authors are aware of the distinction but chose base64 serialization for eviction/restoration contexts.

### Code Paths Examined

- `internal/transform/ledger.go:213-224` — `transformLedgerKeys()` uses `xdr.MarshalBase64(key)` and stores result in `ledgerKeysHash[i]`
- `internal/transform/restored_key.go:22-26` — `TransformRestoredKey()` uses identical `xdr.MarshalBase64(key)` for its `LedgerKeyHash` field
- `internal/transform/schema.go:530-536` — `ContractDataOutput` has both `LedgerKeyHash` (SHA-256) and `LedgerKeyHashBase64` (base64 XDR) as separate columns
- `internal/transform/schema.go:679-686` — `RestoredKeyOutput.LedgerKeyHash` holds base64 XDR, same format as evicted keys
- `internal/utils/main.go:1043-1049` — `LedgerKeyToLedgerKeyHash()` produces SHA-256 hex digests, used only in operation details
- `internal/transform/restored_key_test.go:86` — test expects `"AAAAAgAAAACI4aa0..."` (base64 XDR), confirming intended behavior
- `internal/transform/ledger_test.go:148` — test expects `"AAAABQECAwQFBgcI..."` (base64 XDR), confirming intended behavior
- `cmd/export_ledger_entry_changes.go:120` — `TransformRestoredKey` is used in production export path

### Why It Failed

The hypothesis describes **working-as-designed behavior** with a misleading column name. Both evicted keys (`transformLedgerKeys`) and restored keys (`TransformRestoredKey`) consistently use `xdr.MarshalBase64()` to serialize ledger keys. The `ContractDataOutput` schema demonstrates the authors explicitly distinguish `ledger_key_hash` (SHA-256) from `ledger_key_hash_base_64` (base64 XDR) when they need both formats. The eviction/restoration code paths were designed to export the full base64-encoded key serialization, not a hash. The column name `evicted_ledger_keys_hash` is misleading, but the code behavior is consistent across sibling entities (evicted keys and restored keys) and confirmed by unit tests that assert the base64 format.

### Lesson Learned

When a column name contains "_hash" but the code uses `MarshalBase64()`, check sibling entities (restored keys, contract data) to determine whether this is a naming convention inconsistency or an actual semantic mismatch. If the same serialization approach is used consistently across related entities with matching test assertions, it is working-as-designed despite the confusing name.
