# H002: Ledger `evicted_ledger_keys_hash` exports base64 ledger keys instead of hashes

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_ledgers.evicted_ledger_keys_hash` should contain hashes of the evicted ledger keys, using the same ledger-key hashing convention as the rest of the ETL. Rows should not store raw base64-encoded ledger-key XDR under a field whose name promises hashed identifiers.

## Mechanism

`transformLedgerKeys()` iterates evicted `xdr.LedgerKey` values, calls `xdr.MarshalBase64(key)`, and appends that base64 payload into `ledgerKeysHash`. The resulting ledger export silently mislabels serialized key material as hashes, which breaks consumers that expect `*_hash` columns to be stable hashed identifiers rather than full XDR encodings.

## Trigger

Run `export_ledgers` on any Soroban-era ledger whose close meta contains evicted keys. The JSON row will contain `evicted_ledger_keys_hash` entries like `AAAABQ...` instead of the hex ledger-key hashes used elsewhere in transform outputs.

## Target Code

- `internal/transform/ledger.go:transformLedgerKeys:213-224` — marshals each evicted key to base64 and stores it in the `hash` slice
- `internal/transform/schema.go:LedgerOutput:34-37` — schema advertises `evicted_ledger_keys_hash` as a hash column
- `internal/utils/main.go:LedgerKeyToLedgerKeyHash:1043-1048` — canonical helper shows the ETL's ledger-key hash format
- `internal/transform/ledger_test.go:147-148` — golden test data currently expects base64 in `EvictedLedgerKeysHash`

## Evidence

The local variable is named `keyHash`, the error message says "could not convert ledger key to hash", and the schema column is named `evicted_ledger_keys_hash`, but the implementation never hashes anything; it only base64-encodes the raw XDR key bytes. This differs from contract-data, contract-code, and operation ledger-key fields, which use explicit hash helpers for `ledger_key_hash` and reserve base64 for separately named XDR/base64 fields.

## Anti-Evidence

The checked-in ledger fixture already expects the base64 payload, so the current behavior is stable rather than accidental at runtime. But there is still no sibling column such as `evicted_ledger_keys_base_64` to justify overloading a `*_hash` field with full serialized keys.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (sibling finding 001-restored-key-ledger-key-hash-is-base64 in reviewed/ covers the related restored_key entity, but this is a distinct field in a distinct table)

### Trace Summary

`transformLedgerKeys()` at `ledger.go:213-225` receives `[]xdr.LedgerKey` from `lcmV1.EvictedKeys` (or `lcmV2.EvictedKeys`) and calls `xdr.MarshalBase64(key)` on each key, storing the base64 string in `ledgerKeysHash`. The result is assigned to `LedgerOutput.EvictedLedgerKeysHash` (schema field `evicted_ledger_keys_hash`). Meanwhile, contract_data and contract_code use `utils.LedgerEntryToLedgerKeyHash()` which does `MarshalBinary()` → `hash.Hash()` → `hex.EncodeToString()` to produce actual SHA256 hex hashes for their `ledger_key_hash` fields. Operations use `utils.LedgerKeyToLedgerKeyHash()` which follows the same hashing convention.

### Code Paths Examined

- `internal/transform/ledger.go:transformLedgerKeys:213-225` — iterates evicted keys, calls `xdr.MarshalBase64(key)`, stores result in `ledgerKeysHash` slice. This produces base64-encoded XDR, not a hash.
- `internal/transform/ledger.go:TransformLedger:73-76` — calls `transformLedgerKeys(lcmV1.EvictedKeys)` and assigns to `outputEvictedKeysHash`.
- `internal/transform/ledger.go:TransformLedger:128` — assigns `outputEvictedKeysHash` to `LedgerOutput.EvictedLedgerKeysHash`.
- `internal/transform/schema.go:LedgerOutput:37` — field `EvictedLedgerKeysHash []string` with JSON tag `evicted_ledger_keys_hash`.
- `internal/utils/main.go:LedgerKeyToLedgerKeyHash:1043-1049` — canonical helper: `MarshalBinary()` → `hash.Hash()` → `hex.EncodeToString()`. Produces hex SHA256 hash.
- `internal/utils/main.go:LedgerEntryToLedgerKeyHash:978-985` — same pattern for LedgerEntry input.
- `internal/transform/contract_data.go:68` — uses `utils.LedgerEntryToLedgerKeyHash()` for its `LedgerKeyHash` field (actual hash).
- `internal/transform/contract_code.go:31` — uses `utils.LedgerEntryToLedgerKeyHash()` for its `LedgerKeyHash` field (actual hash).
- `internal/transform/operation.go:1859-1873` — uses `utils.LedgerKeyToLedgerKeyHash()` for operation `ledger_key_hash` details (actual hash).
- `internal/transform/contract_data.go:155` — contract_data also has a separate `LedgerKeyHashBase64` field for base64-encoded XDR, correctly distinguished from the hex hash.

### Findings

The codebase has two distinct conventions for `ledger_key_hash` fields:

1. **contract_data, contract_code, operations**: `ledger_key_hash` = hex-encoded SHA256 hash (e.g., `"abfc33272095a9df4c310cff189040192a8aee6f6a23b6b462889114d80728ca"`). Where base64 is also needed, it's stored in a separate `ledger_key_hash_base_64` field.

2. **ledger evicted keys**: `evicted_ledger_keys_hash` = base64-encoded raw XDR (e.g., `"AAAABQ..."`). No actual hashing is performed.

This means:
- Downstream consumers cannot join `evicted_ledger_keys_hash` with `contract_data.ledger_key_hash` or `contract_code.ledger_key_hash` — they use incompatible formats for the same logical concept.
- Any analytics query that filters or groups by `*_hash` columns expecting fixed-length hex strings will get variable-length base64 blobs instead.
- The semantic contract of the `_hash` suffix is violated: consumers expect deterministic, fixed-length identifiers but get full XDR payloads that vary in length by key type.

### PoC Guidance

- **Test file**: `internal/transform/ledger_test.go`
- **Setup**: Create a test `xdr.LedgerKey` (e.g., a ContractData key). Compute its expected hash using `utils.LedgerKeyToLedgerKeyHash()`. Then build a `[]xdr.LedgerKey` slice containing it.
- **Steps**: Call `transformLedgerKeys()` with the slice. Also call `utils.LedgerKeyToLedgerKeyHash()` on the same key.
- **Assertion**: Assert that `transformLedgerKeys()` returns the same hex hash string as `utils.LedgerKeyToLedgerKeyHash()`. Currently this will fail — `transformLedgerKeys` returns base64 XDR while the utility returns a hex SHA256 hash. This demonstrates the inconsistency.
