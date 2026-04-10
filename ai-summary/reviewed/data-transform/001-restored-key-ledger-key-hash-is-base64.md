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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformRestoredKey()` (restored_key.go:26) calls `xdr.MarshalBase64(key)` to produce the value stored in `RestoredKeyOutput.LedgerKeyHash`. This produces a base64-encoded XDR ledger key payload (e.g., `"AAAAAgAAAACI4aa0pXFSj6qfJuIObLw/5zyugLRGYwxb7wFSr3B9eAAAAAAPiaMn"`), not a hash. In contrast, `TransformContractData()` (contract_data.go:68) and `TransformContractCode()` (contract_code.go:31) both call `utils.LedgerEntryToLedgerKeyHash()` for their `LedgerKeyHash` field, which marshals to binary, SHA-256 hashes the bytes, and hex-encodes the result — a completely different format. Any downstream join on `ledger_key_hash` across these tables will produce zero matches.

### Code Paths Examined

- `internal/transform/restored_key.go:TransformRestoredKey:22-26` — extracts `ledgerKey` via `ledgerEntry.LedgerKey()`, then calls `xdr.MarshalBase64(key)` and assigns the result to `ledgerKeyHash`
- `internal/transform/restored_key.go:40-41` — assigns that base64 string to `RestoredKeyOutput.LedgerKeyHash`
- `internal/utils/main.go:LedgerEntryToLedgerKeyHash:978-985` — the canonical helper: `ledgerKey.MarshalBinary()` → `hash.Hash()` (SHA-256) → `hex.EncodeToString()` — produces a 64-char hex string
- `internal/utils/main.go:LedgerKeyToLedgerKeyHash:1043-1048` — identical logic taking a `LedgerKey` directly, same hex hash output
- `internal/transform/contract_data.go:68` — uses `utils.LedgerEntryToLedgerKeyHash(ledgerEntry)` for `LedgerKeyHash`
- `internal/transform/contract_data.go:75` — uses `xdr.MarshalBase64(ledgerKey)` separately for `LedgerKeyHashBase64`
- `internal/transform/contract_code.go:31` — uses `utils.LedgerEntryToLedgerKeyHash(ledgerEntry)` for `LedgerKeyHash`
- `internal/transform/contract_code.go:38` — uses `xdr.MarshalBase64(ledgerKey)` separately for `LedgerKeyHashBase64`
- `internal/transform/schema.go:679-686` — `RestoredKeyOutput` has only `LedgerKeyHash`, no `LedgerKeyHashBase64` field
- `internal/transform/restored_key_test.go:86` — test fixture expects `"AAAAAgAAAACI4aa0pXFSj6qfJuIObLw/5zyugLRGYwxb7wFSr3B9eAAAAAAPiaMn"` — a base64 string, not a hex hash

### Findings

The bug is clear and the inconsistency is structural:

1. **Format mismatch**: `restored_key.ledger_key_hash` contains base64-encoded XDR (variable-length, contains `+`, `/`, `=` characters). All other `ledger_key_hash` columns contain a 64-character lowercase hex SHA-256 hash.

2. **Semantic violation**: The column name `ledger_key_hash` implies a hash digest, but the restored-key transformer stores the full raw key payload. This is not just a naming issue — it means the column's data contract is violated.

3. **Cross-table join breakage**: Any BigQuery query that joins `restored_key.ledger_key_hash = contract_data.ledger_key_hash` will always return zero rows, even for the same underlying ledger key, because the formats are incompatible.

4. **Missing companion column**: `ContractDataOutput` and `ContractCodeOutput` both have a `LedgerKeyHashBase64` field for the base64 form. `RestoredKeyOutput` has no such field, so the base64 value is being stored in the wrong column rather than in a dedicated one.

5. **Test fixture locks in the bug**: `restored_key_test.go:86` asserts the base64 value, so CI will pass with the current incorrect behavior.

### PoC Guidance

- **Test file**: `internal/transform/restored_key_test.go`
- **Setup**: Use the existing `makeRestoredKeyTestInput()` fixture which produces a valid restored ledger entry change
- **Steps**:
  1. Call `TransformRestoredKey()` with the test input to get the `RestoredKeyOutput`
  2. Also extract the ledger key from the same input and compute the expected hash via `utils.LedgerKeyToLedgerKeyHash(ledgerKey)`
  3. Compare the two values
- **Assertion**: Assert that `output.LedgerKeyHash` equals the hex SHA-256 hash from `utils.LedgerKeyToLedgerKeyHash()`. The current code will fail this assertion because it returns a base64 string instead. Additionally, verify the base64 value contains characters outside `[0-9a-f]` (like `A`, `+`, `/`, `=`) to confirm it is not a hex hash.
