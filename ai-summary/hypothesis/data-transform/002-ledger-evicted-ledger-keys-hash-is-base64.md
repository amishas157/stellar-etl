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
