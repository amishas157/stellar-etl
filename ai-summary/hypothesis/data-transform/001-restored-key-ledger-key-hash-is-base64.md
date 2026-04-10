# H001: Restored-key `ledger_key_hash` exports the full ledger key instead of its hash

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`restored_key.ledger_key_hash` should contain the same hash representation used elsewhere in the ETL for ledger-key identifiers: the hex-encoded SHA-256 of the XDR ledger key bytes. A restored key row should not expose the full base64 ledger-key payload under a field name that claims to be a hash.

## Mechanism

`TransformRestoredKey()` calls `xdr.MarshalBase64(key)` and stores that base64 string directly in `RestoredKeyOutput.LedgerKeyHash`. That deviates from the ETL's other ledger-key outputs, which compute a real hash via `utils.LedgerEntryToLedgerKeyHash()` / `utils.LedgerKeyToLedgerKeyHash()`, so restored-key exports silently publish the wrong identifier format under the same semantic column name.

## Trigger

Run `export_ledger_entry_changes --export-restored-keys` on any ledger range containing `LedgerEntryRestored` changes. The emitted JSON row's `ledger_key_hash` will be a base64-encoded XDR ledger key like `AAAAAg...` instead of the hex hash format used by contract-data and contract-code rows.

## Target Code

- `internal/transform/restored_key.go:TransformRestoredKey:12-46` — computes `LedgerKeyHash` with `xdr.MarshalBase64(key)` and writes it to the output row
- `internal/utils/main.go:LedgerEntryToLedgerKeyHash:978-985` — canonical helper hashes ledger-key bytes and hex-encodes the result
- `internal/utils/main.go:LedgerKeyToLedgerKeyHash:1043-1048` — sibling helper shows the intended ledger-key-hash format
- `internal/transform/schema.go:RestoredKeyOutput:679-686` — schema exposes only `ledger_key_hash`, not a separate base64 ledger-key column
- `internal/transform/restored_key_test.go:84-91` — fixture currently locks in a base64 `ledger_key_hash` value

## Evidence

The restored-key transformer is the only ledger-key export path in `internal/transform/` that names a column `ledger_key_hash` yet fills it with `xdr.MarshalBase64(key)`. Nearby contract transforms compute both forms explicitly: `ContractDataOutput` and `ContractCodeOutput` use `utils.LedgerEntryToLedgerKeyHash()` for `ledger_key_hash` and a separate `ledger_key_hash_base_64` column for the base64 XDR key.

## Anti-Evidence

The existing restored-key test fixture expects the base64 value, so this behavior is currently intentional enough to be regression-tested. But that fixture also highlights the likely data-contract bug: unlike contract-data and contract-code, restored keys have no parallel `ledger_key_hash_base_64` column that would justify storing the full key bytes under a `*_hash` name.
