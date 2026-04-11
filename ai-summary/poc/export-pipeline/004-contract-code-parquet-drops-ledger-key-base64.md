# H004a: Contract-code parquet drops `ledger_key_hash_base_64`

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --write-parquet` exports `contract_code`, parquet rows should preserve both ledger-key identifiers that the JSON path already emits: `ledger_key_hash` and `ledger_key_hash_base_64`. Rows with `ledger_key_hash_base_64="AAAABw..."` in JSON should expose those same values in parquet.

## Mechanism

`TransformContractCode()` computes `LedgerKeyHashBase64`, and the corresponding JSON tests assert it. But the parquet struct and `ToParquet()` converter for `ContractCodeOutput` have no destination for that field. The parquet writer for `ContractCodeOutputParquet` initializes successfully, so when `export_ledger_entry_changes` writes parquet batches for contract_code, the base64 ledger-key payload is silently dropped even though the JSON export for the same rows still contains it.

## Trigger

Run `export_ledger_entry_changes --write-parquet` on any ledger range containing `contract_code` changes. The JSON batch includes `ledger_key_hash_base_64`, while the parquet artifact has no such column.

## Target Code

- `internal/transform/contract_code.go:TransformContractCode:28-41,79-99` — computes and emits `LedgerKeyHashBase64`
- `internal/transform/schema.go:ContractCodeOutput:540-560` — JSON schema includes `ledger_key_hash_base_64`
- `internal/transform/contract_code_test.go:103-124` — expected contract-code output includes `LedgerKeyHashBase64`
- `internal/transform/schema_parquet.go:ContractCodeOutputParquet:284-303` — parquet schema omits the field
- `internal/transform/parquet_converter.go:ContractCodeOutput.ToParquet:316-336` — no mapping for the field
- `cmd/export_ledger_entry_changes.go:exportTransformedData:340-343` — command writes parquet for contract_code (writer init succeeds)

## Evidence

`ledger_key_hash_base_64` is not dead JSON-only baggage: the contract-code transform actively computes it from the live ledger key, and the test fixtures treat it as part of the expected export shape. The parquet writer initializes successfully for `ContractCodeOutputParquet`, so rows are actually written — but without the base64 field.

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

Traced the full path from XDR ledger key extraction through JSON output and parquet serialization. `TransformContractCode()` (contract_code.go:28-41) computes `ledgerKeyHashBase64` via `xdr.MarshalBase64(ledgerKey)` and assigns it to the output struct. The JSON schema struct (`ContractCodeOutput` at schema.go:560) includes the `LedgerKeyHashBase64` field. However, the parquet schema struct (`ContractCodeOutputParquet` at schema_parquet.go:284-303) completely omits this field, and the `ToParquet()` converter method (parquet_converter.go:316-336) has no mapping for it.

### Code Paths Examined

- `internal/transform/contract_code.go:28-41` — computes `ledgerKeyHashBase64` from `xdr.MarshalBase64(ledgerKey)`, assigns to output struct at line 98
- `internal/transform/schema.go:560` — `ContractCodeOutput.LedgerKeyHashBase64` field present with JSON tag `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:284-303` — `ContractCodeOutputParquet` has 18 fields; `LedgerKeyHashBase64` is absent (last field is `NDataSegmentBytes`)
- `internal/transform/parquet_converter.go:316-336` — `ContractCodeOutput.ToParquet()` maps all fields except `LedgerKeyHashBase64`
- `internal/transform/contract_code_test.go:123` — test expects `LedgerKeyHashBase64` populated with a real base64 value
- `cmd/export_ledger_entry_changes.go:340-343` — `ContractCodeOutput` is routed to `ContractCodeOutputParquet` for parquet writing

### Findings

The bug is confirmed. `LedgerKeyHashBase64` is actively computed by the transform function and tested in the test suite. The JSON output path includes the field. The parquet writer initializes successfully for `ContractCodeOutputParquet`, so the export pipeline writes parquet rows — but silently drops this field because:
1. The parquet struct does not define the field
2. The `ToParquet()` method does not map the field

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go`
- **Setup**: Create a `ContractCodeOutput` with non-empty `LedgerKeyHashBase64` value
- **Steps**: Call `.ToParquet()`, verify via reflection that the parquet struct lacks the field; also verify the parquet writer initializes successfully
- **Assertion**: The parquet struct has no `LedgerKeyHashBase64` field, AND the writer init succeeds (confirming this is a live pipeline bug)

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractCodeParquetDropsLedgerKeyHashBase64"
**Test Language**: Go

### Demonstration

The test confirms that `LedgerKeyHashBase64` exists on `ContractCodeOutput` (JSON schema) but is completely absent from `ContractCodeOutputParquet`. When `ToParquet()` is called, the base64-encoded ledger key is silently discarded. Critically, the test also verifies that the parquet writer initializes successfully for `ContractCodeOutputParquet`, proving this is a live export-pipeline bug — rows are written to parquet without the base64 ledger key.

### Test Body

```go
package transform

import (
	"os"
	"reflect"
	"testing"
	"time"

	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/writer"
)

// TestContractCodeParquetDropsLedgerKeyHashBase64 demonstrates that
// ContractCodeOutput.ToParquet() silently drops the LedgerKeyHashBase64 field
// because ContractCodeOutputParquet has no corresponding field.
// In the export pipeline, the contract_code parquet writer initializes successfully,
// so rows are actually written — but without the base64 ledger key.
// A PASSING test confirms the bug.
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
	if fieldExists(parquetType, "LedgerKeyHashBase64") {
		t.Fatal("Expected LedgerKeyHashBase64 to be MISSING from ContractCodeOutputParquet, but it was found — bug may be fixed")
	}

	// 4. Confirm: JSON struct has the field, parquet struct does not
	jsonType := reflect.TypeOf(input)
	if !fieldExists(jsonType, "LedgerKeyHashBase64") {
		t.Fatal("Expected LedgerKeyHashBase64 to exist on ContractCodeOutput JSON struct")
	}

	// 5. Verify the parquet writer can actually initialize for contract_code
	// (confirming this is a live export-pipeline bug, not just a dead schema issue)
	tmpFile, err := os.CreateTemp("", "contract_code_parquet_*.parquet")
	if err != nil {
		t.Fatal("failed to create temp file:", err)
	}
	tmpPath := tmpFile.Name()
	tmpFile.Close()
	defer os.Remove(tmpPath)

	pf, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatal("failed to create parquet file writer:", err)
	}
	pw, err := writer.NewParquetWriter(pf, new(ContractCodeOutputParquet), 1)
	if err != nil {
		pf.Close()
		t.Fatal("parquet writer init failed for ContractCodeOutputParquet — unexpected:", err)
	}
	pw.WriteStop()
	pf.Close()

	t.Logf("BUG CONFIRMED: ContractCodeOutput.LedgerKeyHashBase64=%q is present in JSON schema "+
		"but ContractCodeOutputParquet has no such field. The parquet writer initializes successfully, "+
		"so rows are written to parquet without the base64 ledger key — data is silently lost in the export pipeline.",
		input.LedgerKeyHashBase64)
}

func fieldExists(t reflect.Type, name string) bool {
	for i := 0; i < t.NumField(); i++ {
		if t.Field(i).Name == name {
			return true
		}
	}
	return false
}
```

### Test Output

```
=== RUN   TestContractCodeParquetDropsLedgerKeyHashBase64
    data_integrity_poc_test.go:70: BUG CONFIRMED: ContractCodeOutput.LedgerKeyHashBase64="AAAABwAAAAEAAAAAAAAAAgAAAA==" is present in JSON schema but ContractCodeOutputParquet has no such field. The parquet writer initializes successfully, so rows are written to parquet without the base64 ledger key — data is silently lost in the export pipeline.
--- PASS: TestContractCodeParquetDropsLedgerKeyHashBase64 (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.715s
```
