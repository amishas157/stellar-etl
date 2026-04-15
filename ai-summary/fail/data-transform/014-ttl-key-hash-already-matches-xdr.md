# H014: `ttls.key_hash` already exports the XDR `TtlEntry.KeyHash`

**Date**: 2026-04-15
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`ttls.key_hash` should export the hash stored on the on-chain TTL ledger entry for the archived/restorable object, using a stable textual encoding.

## Mechanism

This looked suspicious because nearby Soroban tables such as `contract_data` and `contract_code` derive `ledger_key_hash` by hashing a full `LedgerKey`, while `TransformTtl()` simply hex-encodes `ttl.KeyHash`. If `ttls.key_hash` were meant to mirror those derived ledger-key hashes, this would silently export the wrong identifier.

## Trigger

Export any TTL ledger-entry change and compare `ttls.key_hash` with the transform's other ledger-key-hash fields.

## Target Code

- `internal/transform/ttl.go:18-40` — reads `ttl.KeyHash` and hex-encodes it
- `internal/transform/schema.go:629-636` — exposes the column as `key_hash`
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:8262-8264` — `TtlEntry` defines only `KeyHash` and `LiveUntilLedgerSeq`

## Evidence

`TransformTtl()` does not call `ledgerEntry.LedgerKey()` or any of the shared ledger-key hash helpers. It exports `ttl.KeyHash.HexString()` directly, which differs from the `ledger_key_hash` generation pattern used by other Soroban entity transforms.

## Anti-Evidence

The XDR type itself stores only `TtlEntry.KeyHash`; there is no full ledger key or alternate hash source on the TTL entry to choose from. The transform is a direct, lossless encoding of the XDR field rather than a remapping decision.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

There is no local transform bug here: `ttls.key_hash` is just the hex form of the exact XDR field on `TtlEntry`. The apparent mismatch comes from comparing two different concepts — a stored TTL hash versus derived ledger-key hashes used by other tables.

### Lesson Learned

When a table is backed by an XDR field that already stores a hash, do not assume it should reuse a helper from a neighboring table with a similar column name. First verify whether the XDR object actually exposes the richer source value needed for that alternate encoding.
