# H023: Soroban contract-data/code Parquet drops `ledger_key_hash_base_64`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When the contract-data and contract-code transforms compute the base64-encoded ledger-key XDR and expose it as `ledger_key_hash_base_64`, the Parquet exports should preserve that same value. The hashed `ledger_key_hash` is not reversible, so dropping the base64 ledger key discards exact source information that JSON still carries.

## Mechanism

Both Soroban ledger-entry transforms call `ledgerEntry.LedgerKey()` plus `xdr.MarshalBase64(...)` and store the result on their JSON/output structs. The Parquet schemas and converters omit `LedgerKeyHashBase64` entirely for both record types, so `export_ledger_entry_changes --write-parquet` silently removes the exact ledger-key encoding from contract-data and contract-code rows while still claiming success.

## Trigger

Run `export_ledger_entry_changes --export-contract-data --write-parquet` or `--export-contract-code --write-parquet` on any range with Soroban contract ledger-entry changes. The JSON rows will contain `ledger_key_hash_base_64`, while the Parquet rows will not have any column for it.

## Target Code

- `internal/transform/contract_data.go:TransformContractData:65-78` — computes the base64-encoded ledger key for contract-data rows
- `internal/transform/contract_data.go:TransformContractData:135-156` — assigns `LedgerKeyHashBase64` onto `ContractDataOutput`
- `internal/transform/contract_code.go:TransformContractCode:28-41` — computes the same base64 ledger key for contract-code rows
- `internal/transform/contract_code.go:TransformContractCode:79-99` — assigns `LedgerKeyHashBase64` onto `ContractCodeOutput`
- `internal/transform/schema.go:ContractDataOutput:515-537` — exposes the contract-data field in the JSON/output schema
- `internal/transform/schema.go:ContractCodeOutput:539-560` — exposes the contract-code field in the JSON/output schema
- `internal/transform/schema_parquet.go:ContractDataOutputParquet:260-281` — defines no Parquet column for the field
- `internal/transform/schema_parquet.go:ContractCodeOutputParquet:283-303` — defines no Parquet column for the field
- `internal/transform/parquet_converter.go:ContractDataOutput.ToParquet:292-313` — omits the field when building Parquet rows
- `internal/transform/parquet_converter.go:ContractCodeOutput.ToParquet:316-336` — omits the field when building Parquet rows
- `cmd/export_ledger_entry_changes.go:exportTransformedData:340-347` — routes both record types into the Parquet writer path

## Evidence

The transforms do real work to materialize `LedgerKeyHashBase64`: they derive the ledger key from XDR, base64-encode it, and store it in the output struct. `internal/transform/contract_data_test.go:150-164` and `internal/transform/contract_code_test.go:112-124` both assert concrete expected values for that field, showing it is part of the intended row shape. Yet neither Parquet schema has anywhere to store it.

## Anti-Evidence

The Parquet rows still retain `ledger_key_hash`, so downstream users keep a lookup hash. But a one-way hash cannot reconstruct the original ledger key bytes, so losing `ledger_key_hash_base_64` is still an irreversible data drop rather than a harmless duplication trim.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Both `TransformContractData` and `TransformContractCode` compute `LedgerKeyHashBase64` by calling `ledgerEntry.LedgerKey()` then `xdr.MarshalBase64(ledgerKey)`, and assign the result to their respective output structs. The JSON schema (`ContractDataOutput`, `ContractCodeOutput`) includes this field and tests assert concrete expected values. However, `ContractDataOutputParquet` and `ContractCodeOutputParquet` have no corresponding field, and both `ToParquet()` converters silently skip the value. This is an irreversible data drop — `ledger_key_hash` is a one-way SHA256 hash that cannot reconstruct the original XDR ledger key bytes.

### Code Paths Examined

- `internal/transform/contract_data.go:65-78` — computes `ledgerKeyHashBase64` via `xdr.MarshalBase64(ledgerKey)`, confirmed populated
- `internal/transform/contract_data.go:135-156` — assigns `LedgerKeyHashBase64: ledgerKeyHashBase64` in `ContractDataOutput` struct literal
- `internal/transform/contract_code.go:28-41` — identical computation for contract-code entries
- `internal/transform/contract_code.go:79-99` — assigns `LedgerKeyHashBase64: ledgerKeyHashBase64` in `ContractCodeOutput` struct literal
- `internal/transform/schema.go:536` — `ContractDataOutput` declares `LedgerKeyHashBase64 string` with JSON tag `ledger_key_hash_base_64`
- `internal/transform/schema.go:560` — `ContractCodeOutput` declares same field
- `internal/transform/schema_parquet.go:260-281` — `ContractDataOutputParquet` has 19 fields; `LedgerKeyHashBase64` is absent
- `internal/transform/schema_parquet.go:283-303` — `ContractCodeOutputParquet` has 18 fields; `LedgerKeyHashBase64` is absent
- `internal/transform/parquet_converter.go:292-313` — `ContractDataOutput.ToParquet()` maps 19 fields; `LedgerKeyHashBase64` not included
- `internal/transform/parquet_converter.go:316-336` — `ContractCodeOutput.ToParquet()` maps 18 fields; `LedgerKeyHashBase64` not included
- `internal/transform/contract_data_test.go:163` — test asserts concrete base64 value for contract-data
- `internal/transform/contract_code_test.go:123` — test asserts concrete base64 value for contract-code

### Findings

The field `LedgerKeyHashBase64` is computed by both transform functions, stored in both JSON output structs, and asserted by unit tests. It carries the base64-encoded XDR representation of the ledger key — the full reversible encoding that allows downstream consumers to reconstruct the exact `LedgerKey` XDR object. The companion field `LedgerKeyHash` is a one-way SHA256 hash useful for lookups but incapable of reconstructing the original key. Both Parquet schemas and both `ToParquet()` converters omit this field entirely, meaning Parquet consumers lose the ability to decode the original ledger key. This follows the same pattern as confirmed findings 001 (contract-event drops operation_id) and 002 (transactions drops tx_signers) in the success directory.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or create alongside existing tests)
- **Setup**: Construct a `ContractDataOutput` with a non-empty `LedgerKeyHashBase64` value (reuse the value from `contract_data_test.go:163`)
- **Steps**: Call `ToParquet()` on the struct, then use reflection to check if the returned `ContractDataOutputParquet` has a field named `LedgerKeyHashBase64`
- **Assertion**: Assert that the Parquet struct either (a) currently lacks the field (demonstrating the bug), or (b) after the fix, contains the field with the same value as the JSON struct. Repeat for `ContractCodeOutput`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractDataParquetDropsLedgerKeyHashBase64" and "TestContractCodeParquetDropsLedgerKeyHashBase64"
**Test Language**: Go

### Demonstration

Both tests construct a JSON output struct with a non-empty `LedgerKeyHashBase64` value, call `ToParquet()`, and use reflection to confirm the returned Parquet struct type has no `LedgerKeyHashBase64` field. This proves the Parquet export path silently discards the base64-encoded ledger key for both contract-data and contract-code rows, while the JSON struct retains it. Since `LedgerKeyHash` is a one-way SHA256 hash, Parquet consumers irreversibly lose the ability to decode the original ledger key XDR.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"
	"time"
)

// TestContractDataParquetDropsLedgerKeyHashBase64 demonstrates that the
// ContractDataOutputParquet struct has no field for LedgerKeyHashBase64,
// so ToParquet() silently drops the value computed by TransformContractData.
func TestContractDataParquetDropsLedgerKeyHashBase64(t *testing.T) {
	// 1. Construct a ContractDataOutput with a non-empty LedgerKeyHashBase64
	cdo := ContractDataOutput{
		ContractId:          "CAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAFCT4",
		ContractKeyType:     "ScValTypeScvLedgerKeyContractInstance",
		ContractDurability:  "ContractDataDurabilityPersistent",
		LastModifiedLedger:  100,
		LedgerEntryChange:   1,
		Deleted:             false,
		ClosedAt:            time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
		LedgerSequence:      100,
		LedgerKeyHash:       "abc123hash",
		LedgerKeyHashBase64: "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAM=", // non-empty value
	}

	// 2. Convert to Parquet via production code path
	parquetRow := cdo.ToParquet()

	// 3. Check that the Parquet struct type has no LedgerKeyHashBase64 field
	parquetType := reflect.TypeOf(parquetRow)
	_, hasField := parquetType.FieldByName("LedgerKeyHashBase64")
	if hasField {
		t.Skip("LedgerKeyHashBase64 field exists in ContractDataOutputParquet — bug may have been fixed")
	}

	// The Parquet struct lacks the field entirely, proving data is silently dropped
	// Verify the JSON struct does have the field and it carries the value
	jsonType := reflect.TypeOf(cdo)
	jsonField, jsonHasField := jsonType.FieldByName("LedgerKeyHashBase64")
	if !jsonHasField {
		t.Fatal("ContractDataOutput unexpectedly missing LedgerKeyHashBase64 field")
	}

	jsonValue := reflect.ValueOf(cdo).FieldByName("LedgerKeyHashBase64").String()
	if jsonValue == "" {
		t.Fatal("LedgerKeyHashBase64 is empty in the JSON struct — test setup error")
	}

	t.Errorf("BUG CONFIRMED: ContractDataOutput has LedgerKeyHashBase64 (json tag: %q, value: %q) "+
		"but ContractDataOutputParquet has no corresponding field — Parquet export silently drops this data",
		jsonField.Tag.Get("json"), jsonValue)
}

// TestContractCodeParquetDropsLedgerKeyHashBase64 demonstrates the same
// silent data drop for ContractCodeOutput → ContractCodeOutputParquet.
func TestContractCodeParquetDropsLedgerKeyHashBase64(t *testing.T) {
	// 1. Construct a ContractCodeOutput with a non-empty LedgerKeyHashBase64
	cco := ContractCodeOutput{
		ContractCodeHash:    "deadbeef",
		ContractCodeExtV:    0,
		LastModifiedLedger:  200,
		LedgerEntryChange:   1,
		Deleted:             false,
		ClosedAt:            time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
		LedgerSequence:      200,
		LedgerKeyHash:       "def456hash",
		LedgerKeyHashBase64: "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAM=", // non-empty value
		NInstructions:       10,
		NFunctions:          5,
	}

	// 2. Convert to Parquet via production code path
	parquetRow := cco.ToParquet()

	// 3. Check that the Parquet struct type has no LedgerKeyHashBase64 field
	parquetType := reflect.TypeOf(parquetRow)
	_, hasField := parquetType.FieldByName("LedgerKeyHashBase64")
	if hasField {
		t.Skip("LedgerKeyHashBase64 field exists in ContractCodeOutputParquet — bug may have been fixed")
	}

	// Verify the JSON struct does have the field
	jsonType := reflect.TypeOf(cco)
	jsonField, jsonHasField := jsonType.FieldByName("LedgerKeyHashBase64")
	if !jsonHasField {
		t.Fatal("ContractCodeOutput unexpectedly missing LedgerKeyHashBase64 field")
	}

	jsonValue := reflect.ValueOf(cco).FieldByName("LedgerKeyHashBase64").String()
	if jsonValue == "" {
		t.Fatal("LedgerKeyHashBase64 is empty in the JSON struct — test setup error")
	}

	t.Errorf("BUG CONFIRMED: ContractCodeOutput has LedgerKeyHashBase64 (json tag: %q, value: %q) "+
		"but ContractCodeOutputParquet has no corresponding field — Parquet export silently drops this data",
		jsonField.Tag.Get("json"), jsonValue)
}
```

### Test Output

```
=== RUN   TestContractDataParquetDropsLedgerKeyHashBase64
    data_integrity_poc_test.go:50: BUG CONFIRMED: ContractDataOutput has LedgerKeyHashBase64 (json tag: "ledger_key_hash_base_64", value: "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAM=") but ContractDataOutputParquet has no corresponding field — Parquet export silently drops this data
--- FAIL: TestContractDataParquetDropsLedgerKeyHashBase64 (0.00s)
=== RUN   TestContractCodeParquetDropsLedgerKeyHashBase64
    data_integrity_poc_test.go:95: BUG CONFIRMED: ContractCodeOutput has LedgerKeyHashBase64 (json tag: "ledger_key_hash_base_64", value: "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAM=") but ContractCodeOutputParquet has no corresponding field — Parquet export silently drops this data
--- FAIL: TestContractCodeParquetDropsLedgerKeyHashBase64 (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.796s
```
