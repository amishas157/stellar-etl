# H002: `extend_footprint_ttl` effects identify TTL wrapper keys instead of the extended Soroban entries

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural identifier mismatch across `history_effects`, `history_operations`, and Soroban state tables
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For an `extend_footprint_ttl` operation, the effect row should identify the same
underlying Soroban entries whose TTL was extended, using an identifier that
matches sibling export surfaces such as `history_operations.details.ledger_key_hash`
and `ttls.key_hash`. A consumer comparing the operation row, the effect row, and
the affected contract-data / contract-code / TTL rows should not have to decode a
different wrapper key format just to refer to the same objects.

## Mechanism

`addExtendFootprintTtlEffect()` walks the changed TTL ledger entries, rebuilds a
new `LedgerKeyTtl{KeyHash}` wrapper for each one, base64-encodes that wrapper,
and exports it under `details.entries`. The sibling operation-details path for the
same operation instead exports the underlying footprint keys via
`ledgerKeyHashFromTxEnvelope()`, and the TTL table itself exports `ttl.key_hash`
as the underlying key hash. As a result, the effect row for the same operation
switches identifier domains from underlying Soroban objects to TTL-entry wrapper
keys, producing plausible but non-joinable `entries` values.

## Trigger

1. Run `export_effects` and `export_operations` over any ledger containing an
   `extend_footprint_ttl` operation that touches at least one contract-data or
   contract-code key.
2. Compare `history_operations.details.ledger_key_hash` with
   `history_effects.details.entries` for the same operation.
3. The operation row will list underlying key hashes, while the effect row will
   contain base64-encoded `LedgerKeyTtl` wrappers around those hashes.

## Target Code

- `internal/transform/effects.go:1443-1474` — `extend_footprint_ttl` effect rebuilds `LedgerKeyTtl` wrappers and base64-encodes them
- `internal/transform/operation.go:1144-1152` — sibling operation-details path exports `ledger_key_hash` from the requested footprint
- `internal/transform/operation.go:1859-1874` — helper returns underlying ledger-key hashes, not TTL wrapper keys
- `internal/transform/ttl.go:28-40` — TTL rows themselves expose the underlying `key_hash`

## Evidence

The effect code never emits the underlying key hash directly; it first converts
`TtlEntry.KeyHash` back into a distinct XDR key type and then serializes that new
wrapper. This differs from the adjacent operation-details and TTL-table exports,
which already expose the affected objects as underlying key hashes.

## Anti-Evidence

The actual ledger entries modified by `extend_footprint_ttl` are TTL rows, and
the current effect tests explicitly assert base64-encoded `LedgerKeyTtl` values.
Review may conclude that `details.entries` is intentionally naming the changed TTL
rows rather than the underlying Soroban objects they govern.
