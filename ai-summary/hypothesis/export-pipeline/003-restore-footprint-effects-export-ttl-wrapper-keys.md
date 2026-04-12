# H003: `restore_footprint` effects identify TTL wrapper keys instead of the restored Soroban entries

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural identifier mismatch across `history_effects`, `history_operations`, and Soroban state tables
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a `restore_footprint` operation, the effect row should identify the restored
underlying Soroban entries in the same identifier space used by sibling exports
(`history_operations.details.ledger_key_hash`, `ttls.key_hash`,
`contract_data.ledger_key_hash`, `contract_code.ledger_key_hash`). A restored-row
consumer should not need to reverse-engineer a TTL-specific wrapper key just to
join the effect back to the restored ledger objects.

## Mechanism

`addRestoreFootprintExpirationEffect()` exports `details.entries` by rebuilding
`LedgerKeyTtl{KeyHash}` for each changed TTL row and base64-encoding that wrapper.
But the sibling operation row for the same restore action exports the requested
footprint as underlying ledger-key hashes, and the TTL / contract-state tables
also identify the restored objects by underlying hashes. The effect therefore
describes the same restore action using a different object namespace, yielding
plausible `entries` values that do not line up directly with the rest of the
export pipeline.

## Trigger

1. Export operations and effects for any ledger containing a `restore_footprint`
   operation that restores one or more contract-data or contract-code entries.
2. Compare the operation row's `details.ledger_key_hash` list to the effect row's
   `details.entries` list.
3. The operation row will contain underlying footprint hashes, while the effect
   row will contain base64-encoded `LedgerKeyTtl` wrapper keys derived from those
   hashes.

## Target Code

- `internal/transform/effects.go:1477-1515` — `restore_footprint` effect rebuilds and exports `LedgerKeyTtl` wrapper keys
- `internal/transform/operation.go:1153-1159` — sibling operation-details path exports footprint `ledger_key_hash` values
- `internal/transform/operation.go:1859-1874` — helper returns underlying key hashes from the Soroban footprint
- `internal/transform/ttl.go:28-40` — restored TTL rows expose the underlying `key_hash`, not the wrapper key

## Evidence

The restore-effect code follows the same pattern as the extend-effect code: it
never emits the underlying hash directly, instead materializing a new TTL ledger
key and serializing that wrapper. That creates a mismatch with the identifier
representation used by the operation row and the restored state tables for the
same footprint.

## Anti-Evidence

As with `extend_footprint_ttl`, reviewers may decide that the effect is supposed
to name the TTL ledger entries that actually changed, not the restored underlying
Soroban entries. The current tests also assert base64 TTL wrapper keys, which
shows this shape is at least deliberate in checked-in fixtures.
