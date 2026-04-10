# H003: Contract-data Parquet omits populated `ledger_key_hash_base_64`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each `contract_data` Parquet row should preserve the same `ledger_key_hash_base_64` value that `TransformContractData()` computes and the JSON export emits. The base64 ledger-key encoding should remain available for downstream consumers that reconstruct or compare the original XDR ledger key.

## Mechanism

`TransformContractData()` marshals the ledger key to base64 and stores it on `ContractDataOutput.LedgerKeyHashBase64`. The Parquet schema and converter never carry that field forward, so `export_ledger_entry_changes --export-contract-data --write-parquet` silently drops a populated identifier column from every non-nonce contract-data row.

## Trigger

Run `export_ledger_entry_changes --export-contract-data --write-parquet` over any ledger that contains contract-data entries. The JSON output will include `ledger_key_hash_base_64`, but the Parquet file will have no such column and therefore no way to retain the computed value.

## Target Code

- `internal/transform/contract_data.go:65-78` — computes `ledgerKeyHashBase64` from the ledger key XDR
- `internal/transform/contract_data.go:135-156` — writes `LedgerKeyHashBase64` into `ContractDataOutput`
- `internal/transform/schema.go:530-536` — JSON schema exposes `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:261-281` — Parquet schema omits `LedgerKeyHashBase64`
- `internal/transform/parquet_converter.go:292-313` — converter omits the populated field
- `cmd/export_ledger_entry_changes.go:212-230` — contract-data rows are produced in the live export path
- `cmd/export_ledger_entry_changes.go:321-347` — Parquet export chooses `ContractDataOutputParquet` for those rows

## Evidence

The transformer explicitly computes both `LedgerKeyHash` and `LedgerKeyHashBase64`, and both are kept on the JSON output struct. But only the hex hash survives the Parquet schema/converter pair, so the base64 representation is deterministically lost whenever the parquet path is used.

## Anti-Evidence

Consumers can still access the hex `ledger_key_hash`, so one identifier remains. But the code intentionally computes and exports the base64 XDR form separately, which indicates the second representation is meant to be preserved rather than silently discarded.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformContractData()` at `contract_data.go:65-78` marshals the ledger key to base64 via `xdr.MarshalBase64(ledgerKey)` and stores it in `ledgerKeyHashBase64`. This value is assigned to `ContractDataOutput.LedgerKeyHashBase64` at line 155. The JSON schema at `schema.go:536` declares the field with tag `json:"ledger_key_hash_base_64"`. However, `ContractDataOutputParquet` (schema_parquet.go:261-281) has no `LedgerKeyHashBase64` field — it ends at `ContractDataXDR` on line 280. The `ToParquet()` converter (parquet_converter.go:292-313) likewise stops at `ContractDataXDR` and never copies `LedgerKeyHashBase64`. This is the same class of bug as the confirmed finding in success/002 (contract event parquet dropping operation_id).

### Code Paths Examined

- `internal/transform/contract_data.go:65-78` — confirmed: computes `ledgerKeyHashBase64` from `xdr.MarshalBase64(ledgerKey)` and errors out if marshaling fails, so the value is always populated for non-nonce rows
- `internal/transform/contract_data.go:135-156` — confirmed: `LedgerKeyHashBase64: ledgerKeyHashBase64` assigned in the output struct literal
- `internal/transform/schema.go:536` — confirmed: `LedgerKeyHashBase64 string` with JSON tag `ledger_key_hash_base_64` is the last field of `ContractDataOutput`
- `internal/transform/schema_parquet.go:260-281` — confirmed: `ContractDataOutputParquet` has 20 fields ending with `ContractDataXDR`; no `LedgerKeyHashBase64` field exists
- `internal/transform/parquet_converter.go:292-313` — confirmed: `ToParquet()` maps all 20 fields but omits `LedgerKeyHashBase64`; the last assignment is `ContractDataXDR: cdo.ContractDataXDR`

### Findings

The bug is a straightforward omission: when `LedgerKeyHashBase64` was added to the JSON schema, neither the Parquet schema struct nor the Parquet converter were updated to carry the field. Every non-nonce contract-data row exported via the Parquet path silently loses its base64 ledger key encoding. The hex `LedgerKeyHash` is preserved, so one identifier survives, but the two are semantically different — the hex is a hash of the ledger key, while the base64 is a serialization of the full ledger key XDR, enabling reconstruction of the original key. Dropping it is data loss.

### PoC Guidance

- **Test file**: `internal/transform/contract_data_test.go` (append a new test)
- **Setup**: Use existing test fixtures to create a `ContractDataOutput` with a populated `LedgerKeyHashBase64` field; alternatively, construct a minimal `ContractDataOutput` struct literal with `LedgerKeyHashBase64` set to a non-empty string
- **Steps**: Call `ToParquet()` on the `ContractDataOutput` and cast the result to `ContractDataOutputParquet`
- **Assertion**: Verify that `ContractDataOutputParquet` has a `LedgerKeyHashBase64` field and that it equals the source value. Under the current code, the Parquet struct lacks the field entirely, so this test will fail to compile — demonstrating that the field is structurally absent from the Parquet schema

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractDataParquetDropsLedgerKeyHashBase64"
**Test Language**: Go

### Demonstration

The test constructs a `ContractDataOutput` with `LedgerKeyHashBase64` set to a non-empty base64 string, then calls `ToParquet()` and uses reflection to check whether the returned `ContractDataOutputParquet` struct contains a `LedgerKeyHashBase64` field. The field is structurally absent from the Parquet schema, confirming that every Parquet export silently drops the base64 ledger key encoding that the JSON path preserves.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"
	"time"
)

// TestContractDataParquetDropsLedgerKeyHashBase64 demonstrates that
// ContractDataOutputParquet omits the LedgerKeyHashBase64 field that
// ContractDataOutput populates, silently dropping data on Parquet export.
func TestContractDataParquetDropsLedgerKeyHashBase64(t *testing.T) {
	// 1. Construct a ContractDataOutput with LedgerKeyHashBase64 populated
	src := ContractDataOutput{
		ContractId:          "CABCDEFG",
		ContractKeyType:     "ScValTypeScvLedgerKeyContractInstance",
		ContractDurability:  "persistent",
		LedgerKeyHash:       "abcdef1234567890",
		LedgerKeyHashBase64: "AAAAIQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
		LastModifiedLedger:  100,
		LedgerEntryChange:   1,
		LedgerSequence:      200,
		ClosedAt:            time.Now(),
		ContractDataXDR:     "deadbeef",
	}

	// Verify the source value is populated
	if src.LedgerKeyHashBase64 == "" {
		t.Fatal("test setup: LedgerKeyHashBase64 should be non-empty")
	}

	// 2. Convert to Parquet via production code path
	parquetRaw := src.ToParquet()

	// 3. Use reflection to check if LedgerKeyHashBase64 survived the conversion
	parquetVal := reflect.ValueOf(parquetRaw)
	parquetType := parquetVal.Type()

	field := findField(parquetType, "LedgerKeyHashBase64")
	if field == nil {
		t.Errorf("ContractDataOutputParquet struct is missing LedgerKeyHashBase64 field entirely — "+
			"the JSON schema has it (ContractDataOutput.LedgerKeyHashBase64 = %q) but the Parquet "+
			"schema drops it, causing silent data loss on every Parquet export",
			src.LedgerKeyHashBase64)
		return
	}

	// If the field exists, verify the value was copied
	got := parquetVal.FieldByName("LedgerKeyHashBase64").String()
	if got != src.LedgerKeyHashBase64 {
		t.Errorf("LedgerKeyHashBase64 not preserved: got %q, want %q", got, src.LedgerKeyHashBase64)
	}
}

func findField(t reflect.Type, name string) *reflect.StructField {
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		if f.Name == name {
			return &f
		}
	}
	return nil
}
```

### Test Output

```
=== RUN   TestContractDataParquetDropsLedgerKeyHashBase64
    data_integrity_poc_test.go:41: ContractDataOutputParquet struct is missing LedgerKeyHashBase64 field entirely — the JSON schema has it (ContractDataOutput.LedgerKeyHashBase64 = "AAAAIQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=") but the Parquet schema drops it, causing silent data loss on every Parquet export
--- FAIL: TestContractDataParquetDropsLedgerKeyHashBase64 (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.813s
```
