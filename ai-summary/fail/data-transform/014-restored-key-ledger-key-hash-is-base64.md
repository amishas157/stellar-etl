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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestRestoredKeyLedgerKeyHashIsBase64NotHexHash"
**Test Language**: Go

### Demonstration

The test calls `TransformRestoredKey()` with the existing test fixture and compares the `LedgerKeyHash` output against the canonical hex SHA-256 hash computed by `utils.LedgerKeyToLedgerKeyHash()`. The output is `"AAAAAgAAAACI4aa0pXFSj6qfJuIObLw/5zyugLRGYwxb7wFSr3B9eAAAAAAPiaMn"` (a base64-encoded XDR key) while the correct hash should be `"8cc323d734b0263b26708e4842c3a3a9cd3db13146e5e7a5862092749e5d3377"` (a 64-char hex SHA-256). The values don't match and the output contains uppercase and special characters confirming it's base64, not a hex hash.

### Test Body

```go
func TestRestoredKeyLedgerKeyHashIsBase64NotHexHash(t *testing.T) {
	// Step 1: Use the existing test fixture to construct a restored ledger entry
	change, err := makeRestoredKeyTestInput()
	if err != nil {
		t.Fatalf("failed to create test input: %v", err)
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	// Step 2: Run the production transform
	output, err := TransformRestoredKey(change, header)
	if err != nil {
		t.Fatalf("TransformRestoredKey failed: %v", err)
	}

	// Step 3: Compute the expected hex SHA-256 hash via the canonical helper
	ledgerEntry := change.Post
	ledgerKey, err := ledgerEntry.LedgerKey()
	if err != nil {
		t.Fatalf("failed to extract ledger key: %v", err)
	}
	expectedHash := utils.LedgerKeyToLedgerKeyHash(ledgerKey)

	// Step 4: Verify the expected hash is a valid 64-char lowercase hex string
	if len(expectedHash) != 64 {
		t.Fatalf("expected hash length 64 (hex SHA-256), got %d: %s", len(expectedHash), expectedHash)
	}
	if _, err := hex.DecodeString(expectedHash); err != nil {
		t.Fatalf("expected hash is not valid hex: %s", expectedHash)
	}

	// Step 5: Demonstrate the bug — output.LedgerKeyHash is NOT the hex hash
	t.Logf("output.LedgerKeyHash = %q", output.LedgerKeyHash)
	t.Logf("expected hex hash    = %q", expectedHash)

	if output.LedgerKeyHash == expectedHash {
		t.Fatal("Expected LedgerKeyHash to differ from the hex hash (bug not present), but they matched")
	}

	// Step 6: Confirm the output is base64 (contains chars outside [0-9a-f])
	hasNonHexChars := false
	for _, c := range output.LedgerKeyHash {
		if !strings.ContainsRune("0123456789abcdef", c) {
			hasNonHexChars = true
			break
		}
	}
	if !hasNonHexChars {
		t.Fatal("Expected LedgerKeyHash to contain non-hex characters (base64), but it looks like hex")
	}

	t.Logf("BUG CONFIRMED: LedgerKeyHash contains base64-encoded XDR (%d chars) instead of hex SHA-256 hash (64 chars)",
		len(output.LedgerKeyHash))
	t.Logf("This means restored_key.ledger_key_hash cannot be joined with contract_data.ledger_key_hash or contract_code.ledger_key_hash")
}
```

### Test Output

```
=== RUN   TestRestoredKeyLedgerKeyHashIsBase64NotHexHash
    data_integrity_poc_test.go:167: output.LedgerKeyHash = "AAAAAgAAAACI4aa0pXFSj6qfJuIObLw/5zyugLRGYwxb7wFSr3B9eAAAAAAPiaMn"
    data_integrity_poc_test.go:168: expected hex hash    = "8cc323d734b0263b26708e4842c3a3a9cd3db13146e5e7a5862092749e5d3377"
    data_integrity_poc_test.go:186: BUG CONFIRMED: LedgerKeyHash contains base64-encoded XDR (64 chars) instead of hex SHA-256 hash (64 chars)
    data_integrity_poc_test.go:188: This means restored_key.ledger_key_hash cannot be joined with contract_data.ledger_key_hash or contract_code.ledger_key_hash
--- PASS: TestRestoredKeyLedgerKeyHashIsBase64NotHexHash (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.721s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-11
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Exercises claimed bug**: YES — the PoC correctly shows that `TransformRestoredKey()` emits base64-encoded ledger-key XDR while `utils.LedgerKeyToLedgerKeyHash()` produces a hex SHA-256 digest for the same key.
2. **Realistic preconditions**: YES — the existing restored-key fixture and `testdata/changes/restored_key.golden` confirm that real `LedgerEntryRestored` exports take this path.
3. **Bug vs by-design**: NOT PROVEN BUG — the checked-in unit test (`internal/transform/restored_key_test.go`) and end-to-end golden output (`testdata/changes/restored_key.golden`) both intentionally assert the base64 form for `restored_key.ledger_key_hash`. I found no repository documentation, schema contract, or sibling code that requires the restored-key table specifically to use the hex digest representation.
4. **Impact vs severity**: NOT HIGH — the claimed downstream join breakage is speculative. The PoC proves only a representation mismatch, not that any consumer receives incorrect restored-key data. At most this supports an informational consistency/naming concern.
5. **In scope**: NO — without proof that the export is semantically wrong rather than intentionally serialized differently, this reduces to a naming/convention complaint, which is out of scope for data-integrity findings.
6. **Test correctness**: PARTIAL — the test is technically sound for proving inequality with the hash helper, but it assumes without evidence that `restored_key.ledger_key_hash` is supposed to equal that helper's output.
7. **Alternative explanations**: PRESENT — `restored_key` may intentionally store the full serialized ledger key because this table has no separate raw-key/base64 column and spans entry types that do not all have sibling tables using the hex-hash convention.
8. **Novelty**: UNRESOLVED — novelty is not relevant to the rejection.

### Rejection Reason

The PoC demonstrates an inconsistency in representation, but it does not establish that the restored-key export is wrong. The repository's own checked-in regression tests and golden output intentionally lock in base64 for this table, and there is no codebase contract showing that `restored_key.ledger_key_hash` must use the hex SHA-256 convention used elsewhere. Without that missing contract, the finding is an unproven naming/consistency issue rather than confirmed data corruption.

### Failed Checks

3, 4, 5, 6, 7
