# H055: `ttls.key_hash` should export contract identifiers instead of the raw TTL hash

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Low
**Impact**: schema/documentation mismatch
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the `ttls.key_hash` column semantically represents a contract code hash or contract ID, then exported TTL rows should preserve one of those higher-level identifiers so downstream consumers can join TTL rows directly to contract-code or contract-data tables. The column should not require readers to reinterpret it as some other opaque hash.

## Mechanism

`TransformTtl()` simply hex-encodes `ttl.KeyHash`, while the schema comment says `key_hash is contract_code_hash or contract_id`. That looked like a plausible data bug because the exported value is a raw 32-byte hash digest, not a contract strkey or contract-code hash string, so the column's documented meaning appears to overpromise a more directly joinable identifier than the code emits.

## Trigger

1. Export any TTL row with `export_ledger_entry_changes --export-ttl`.
2. Compare the `key_hash` output with a contract ID or contract-code hash derived from the underlying archived entry.
3. Observe that the exported column is just the hex form of `TtlEntry.KeyHash`.

## Target Code

- `internal/transform/schema.go:629-632` — comment says `key_hash is contract_code_hash or contract_id`
- `internal/transform/ttl.go:28-40` — exporter uses `ttl.KeyHash.HexString()` directly
- `internal/transform/ttl_test.go:97-104` — tests assert the raw hash hex output

## Evidence

The code path is straightforward: `TransformTtl()` reads `ttl.KeyHash` from the XDR entry and writes its hex string into `TtlOutput.KeyHash`, and the unit test expects exactly that raw hash. The only hint that something richer might be intended is the schema comment that describes the field as a contract code hash or contract ID.

## Anti-Evidence

The XDR `TtlEntry` exposes only `KeyHash`; it does not carry the original contract-data / contract-code ledger key, contract ID, or contract-code hash in reversible form. Since the transform already exports the only on-chain identifier present on the TTL entry, there is no alternative source value the current code could serialize instead.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is a misleading comment, not corrupt exported data. `TransformTtl()` correctly preserves the raw `TtlEntry.KeyHash` field that exists on-chain, and the repo's tests explicitly lock in that behavior; there is no separate contract ID or contract-code hash available to export from a TTL row.

### Lesson Learned

TTL rows only expose the opaque `KeyHash` carried by `TtlEntry`; they do not retain enough information to reconstruct the underlying contract identifier. A comment that over-interprets that hash is not enough for a correctness finding unless the XDR actually provides a different value the exporter is dropping.
