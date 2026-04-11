# H004: Ledger-entry-change parquet drops `ledger_key_hash_base_64` for contract data and code

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --write-parquet` exports `contract_data` or `contract_code`, parquet rows should preserve both ledger-key identifiers that the JSON path already emits: `ledger_key_hash` and `ledger_key_hash_base_64`. For the repository fixtures, rows with `ledger_key_hash_base_64="AAAABg..."` or `"AAAABw..."` in JSON should expose those same values in parquet.

## Mechanism

Both `TransformContractData()` and `TransformContractCode()` compute `LedgerKeyHashBase64`, and the corresponding JSON tests assert it. But the parquet structs and `ToParquet()` converters for `ContractDataOutput` and `ContractCodeOutput` have no destination for that field. As soon as `export_ledger_entry_changes` writes parquet batches for these resource types, the base64 ledger-key payload disappears even though the JSON export for the same rows still contains it.

## Trigger

Run `export_ledger_entry_changes --write-parquet` on any ledger range containing `contract_data` or `contract_code` changes. The JSON batch includes `ledger_key_hash_base_64`, while the parquet artifact has no such column for those entity types.

## Target Code

- `internal/transform/contract_data.go:TransformContractData:65-78,135-156` — computes and emits `LedgerKeyHashBase64`
- `internal/transform/contract_code.go:TransformContractCode:28-41,79-99` — computes and emits `LedgerKeyHashBase64`
- `internal/transform/schema.go:ContractDataOutput:516-537` — JSON schema includes `ledger_key_hash_base_64`
- `internal/transform/schema.go:ContractCodeOutput:540-560` — JSON schema includes `ledger_key_hash_base_64`
- `internal/transform/contract_data_test.go:142-164` — expected contract-data output includes `LedgerKeyHashBase64`
- `internal/transform/contract_code_test.go:103-124` — expected contract-code output includes `LedgerKeyHashBase64`
- `internal/transform/schema_parquet.go:ContractDataOutputParquet:261-281` — parquet schema omits the field
- `internal/transform/schema_parquet.go:ContractCodeOutputParquet:284-303` — parquet schema omits the field
- `internal/transform/parquet_converter.go:ContractDataOutput.ToParquet:292-313` — no mapping for the field
- `internal/transform/parquet_converter.go:ContractCodeOutput.ToParquet:316-336` — no mapping for the field
- `cmd/export_ledger_entry_changes.go:exportTransformedData:321-347,370-372` — command writes parquet for both resource types

## Evidence

`ledger_key_hash_base_64` is not dead JSON-only baggage: both contract-data and contract-code transforms actively compute it from the live ledger key, and both test fixtures treat it as part of the expected export shape. The parquet path is the only place where it disappears.

## Anti-Evidence

`ledger_key_hash` is still present, so parquet consumers retain one identifier for the row. But the missing base64 field contains the serialized ledger-key form already exposed by JSON, so parquet users lose information that cannot be recovered from the hash alone.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full path from XDR ledger key extraction through JSON output and parquet serialization. Both `TransformContractData()` (contract_data.go:65-78) and `TransformContractCode()` (contract_code.go:28-41) compute `ledgerKeyHashBase64` via `xdr.MarshalBase64(ledgerKey)` and assign it to the output struct. The JSON schema structs (`ContractDataOutput` at schema.go:536, `ContractCodeOutput` at schema.go:560) include the `LedgerKeyHashBase64` field. However, both parquet schema structs (`ContractDataOutputParquet` at schema_parquet.go:261-281, `ContractCodeOutputParquet` at schema_parquet.go:284-303) completely omit this field, and both `ToParquet()` converter methods (parquet_converter.go:292-313 and 316-336) have no mapping for it.

### Code Paths Examined

- `internal/transform/contract_data.go:65-78` — computes `ledgerKeyHashBase64` from `xdr.MarshalBase64(ledgerKey)`, assigns to output struct at line 155
- `internal/transform/contract_code.go:28-41` — same computation, assigns to output struct at line 98
- `internal/transform/schema.go:536` — `ContractDataOutput.LedgerKeyHashBase64` field present with JSON tag `ledger_key_hash_base_64`
- `internal/transform/schema.go:560` — `ContractCodeOutput.LedgerKeyHashBase64` field present with JSON tag `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:261-281` — `ContractDataOutputParquet` has 20 fields; `LedgerKeyHashBase64` is absent (last field is `ContractDataXDR`)
- `internal/transform/schema_parquet.go:284-303` — `ContractCodeOutputParquet` has 18 fields; `LedgerKeyHashBase64` is absent (last field is `NDataSegmentBytes`)
- `internal/transform/parquet_converter.go:292-313` — `ContractDataOutput.ToParquet()` maps all fields except `LedgerKeyHashBase64`
- `internal/transform/parquet_converter.go:316-336` — `ContractCodeOutput.ToParquet()` maps all fields except `LedgerKeyHashBase64`
- `internal/transform/contract_data_test.go:163` — test expects `LedgerKeyHashBase64` populated with a real base64 value
- `internal/transform/contract_code_test.go:123` — test expects `LedgerKeyHashBase64` populated with a real base64 value

### Findings

The bug is confirmed. `LedgerKeyHashBase64` is actively computed by both transform functions and tested in both test suites. The JSON output path includes the field. But the parquet output path silently drops it because:
1. Neither parquet struct defines the field
2. Neither `ToParquet()` method maps the field

This is Pattern 2 (Parquet field mapping omission). The field contains the base64-serialized ledger key, which is distinct from `LedgerKeyHash` (an FNV hash). `LedgerKeyHash` is present in parquet, but `LedgerKeyHashBase64` cannot be recovered from the hash — they carry different information. Parquet consumers lose the ability to deserialize the original ledger key.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or nearest parquet conversion test)
- **Setup**: Create a `ContractDataOutput` and `ContractCodeOutput` with non-empty `LedgerKeyHashBase64` values (use values from existing test fixtures: `"AAAABg..."` and `"AAAABw..."`)
- **Steps**: Call `.ToParquet()` on each, cast result to the parquet struct type
- **Assertion**: Assert that the returned parquet struct contains a `LedgerKeyHashBase64` field with the same value. Currently this will fail because the field does not exist on the parquet struct. The fix requires adding `LedgerKeyHashBase64 string` to both `ContractDataOutputParquet` and `ContractCodeOutputParquet` in `schema_parquet.go`, and adding the field mapping in both `ToParquet()` methods in `parquet_converter.go`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractDataParquetDropsLedgerKeyHashBase64" and "TestContractCodeParquetDropsLedgerKeyHashBase64"
**Test Language**: Go

### Demonstration

Both tests confirm that `LedgerKeyHashBase64` exists on the JSON output structs (`ContractDataOutput` and `ContractCodeOutput`) but is completely absent from their parquet counterparts (`ContractDataOutputParquet` and `ContractCodeOutputParquet`). When `ToParquet()` is called, the base64-encoded ledger key is silently discarded because the destination struct has no field to receive it. This means any parquet export of contract data or contract code loses the serialized ledger key that JSON exports preserve.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"
	"time"
)

// TestContractDataParquetDropsLedgerKeyHashBase64 demonstrates that
// ContractDataOutput.ToParquet() silently drops the LedgerKeyHashBase64 field
// because ContractDataOutputParquet has no corresponding field.
// A PASSING test confirms the bug: parquet output loses the base64 ledger key.
func TestContractDataParquetDropsLedgerKeyHashBase64(t *testing.T) {
	// 1. Construct input with a non-empty LedgerKeyHashBase64
	input := ContractDataOutput{
		ContractId:          "CABCDEFGHIJKLMNOPQRSTUVWXYZ234567ABCDEFGHIJKLMNOPQRSTUV",
		LedgerKeyHash:       "abc123hash",
		LedgerKeyHashBase64: "AAAABgAAAAEAAAAAAAAAAgAAAA==",
		LedgerSequence:      100,
		ClosedAt:            time.Now(),
	}

	if input.LedgerKeyHashBase64 == "" {
		t.Fatal("precondition: LedgerKeyHashBase64 must be set on input")
	}

	// 2. Run the production ToParquet() converter
	parquetResult := input.ToParquet()

	// 3. Assert the parquet struct is missing the LedgerKeyHashBase64 field
	parquetType := reflect.TypeOf(parquetResult)
	field := findField(parquetType, "LedgerKeyHashBase64")
	if field != nil {
		t.Fatal("Expected LedgerKeyHashBase64 to be MISSING from parquet struct, but it was found — bug may be fixed")
	}

	// 4. Confirm: JSON struct has the field, parquet struct does not — data is silently lost
	jsonType := reflect.TypeOf(input)
	jsonField := findField(jsonType, "LedgerKeyHashBase64")
	if jsonField == nil {
		t.Fatal("Expected LedgerKeyHashBase64 to exist on JSON struct")
	}

	t.Logf("BUG CONFIRMED: ContractDataOutput.LedgerKeyHashBase64=%q is present in JSON schema "+
		"but ContractDataOutputParquet has no such field — value is silently dropped by ToParquet()",
		input.LedgerKeyHashBase64)
}

// TestContractCodeParquetDropsLedgerKeyHashBase64 demonstrates that
// ContractCodeOutput.ToParquet() silently drops the LedgerKeyHashBase64 field
// because ContractCodeOutputParquet has no corresponding field.
// A PASSING test confirms the bug: parquet output loses the base64 ledger key.
func TestContractCodeParquetDropsLedgerKeyHashBase64(t *testing.T) {
	// 1. Construct input with a non-empty LedgerKeyHashBase64
	input := ContractCodeOutput{
		ContractCodeHash:    "deadbeef",
		LedgerKeyHash:       "abc123hash",
		LedgerKeyHashBase64: "AAAABwAAAAEAAAAAAAAAAgAAAA==",
		LedgerSequence:      200,
		ClosedAt:            time.Now(),
	}

	if input.LedgerKeyHashBase64 == "" {
		t.Fatal("precondition: LedgerKeyHashBase64 must be set on input")
	}

	// 2. Run the production ToParquet() converter
	parquetResult := input.ToParquet()

	// 3. Assert the parquet struct is missing the LedgerKeyHashBase64 field
	parquetType := reflect.TypeOf(parquetResult)
	field := findField(parquetType, "LedgerKeyHashBase64")
	if field != nil {
		t.Fatal("Expected LedgerKeyHashBase64 to be MISSING from parquet struct, but it was found — bug may be fixed")
	}

	// 4. Confirm: JSON struct has the field, parquet struct does not — data is silently lost
	jsonType := reflect.TypeOf(input)
	jsonField := findField(jsonType, "LedgerKeyHashBase64")
	if jsonField == nil {
		t.Fatal("Expected LedgerKeyHashBase64 to exist on JSON struct")
	}

	t.Logf("BUG CONFIRMED: ContractCodeOutput.LedgerKeyHashBase64=%q is present in JSON schema "+
		"but ContractCodeOutputParquet has no such field — value is silently dropped by ToParquet()",
		input.LedgerKeyHashBase64)
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
    data_integrity_poc_test.go:44: BUG CONFIRMED: ContractDataOutput.LedgerKeyHashBase64="AAAABgAAAAEAAAAAAAAAAgAAAA==" is present in JSON schema but ContractDataOutputParquet has no such field — value is silently dropped by ToParquet()
--- PASS: TestContractDataParquetDropsLedgerKeyHashBase64 (0.00s)
=== RUN   TestContractCodeParquetDropsLedgerKeyHashBase64
    data_integrity_poc_test.go:84: BUG CONFIRMED: ContractCodeOutput.LedgerKeyHashBase64="AAAABwAAAAEAAAAAAAAAAgAAAA==" is present in JSON schema but ContractCodeOutputParquet has no such field — value is silently dropped by ToParquet()
--- PASS: TestContractCodeParquetDropsLedgerKeyHashBase64 (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.921s
```

---

## Final Review — Needs Revision

**Date**: 2026-04-11
**Final review by**: gpt-5.4, high

### What Needs Fixing

The `contract_code` half is real, but the `contract_data` half is overclaimed for the **export-pipeline** framing. `ContractDataOutput.ToParquet()` does drop `LedgerKeyHashBase64`, but `export_ledger_entry_changes --export-contract-data --write-parquet` does not currently emit contract-data parquet rows at all because `writer.NewParquetWriter(..., new(ContractDataOutputParquet), ...)` fails immediately with `type MAP: not a valid Type string` from the `Key`/`KeyDecoded`/`Val`/`ValDecoded` tags.

That means the current PoC proves a transform/schema omission for `contract_data`, but not the claimed pipeline behavior of silently emitting rows with that column missing. The `contract_code` path does initialize and write, so that half still fits the original framing.

### Revision Instructions

1. Split this into two findings instead of one combined document.
2. Keep **contract_code** in `export-pipeline`: the live parquet path initializes successfully and drops populated `ledger_key_hash_base_64`.
3. Move **contract_data** to a narrower framing, or explicitly account for the writer-init blocker. As the code stands today, the stronger export-pipeline bug for `contract_data` is the separate schema-initialization failure, not row-level loss of `ledger_key_hash_base_64`.
4. If you keep a `contract_data` omission finding, scope it to the transform/parquet conversion layer unless you first demonstrate a production path that actually serializes contract-data parquet rows.

### Checks Passed So Far

- `TransformContractData()` and `TransformContractCode()` both populate non-empty `LedgerKeyHashBase64` values.
- `ContractDataOutputParquet` and `ContractCodeOutputParquet` both omit a `LedgerKeyHashBase64` destination field, so `ToParquet()` cannot preserve the value.
- `export_ledger_entry_changes` routes both resource types through `WriteParquet()`.
- `ContractCodeOutputParquet` writer initialization succeeds, so the `contract_code` export-pipeline bug is viable.
- No evidence suggests the omission is intentional or by design.
