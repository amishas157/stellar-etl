# H023: Soroban contract-code Parquet drops `ledger_key_hash_base_64`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When the contract-code transform computes the base64-encoded ledger-key XDR and exposes it as `ledger_key_hash_base_64`, the Parquet export should preserve that same value. The hashed `ledger_key_hash` is not reversible, so dropping the base64 ledger key discards exact source information that JSON still carries.

## Mechanism

The contract-code transform calls `ledgerEntry.LedgerKey()` plus `xdr.MarshalBase64(...)` and stores the result on the JSON output struct (`ContractCodeOutput.LedgerKeyHashBase64`). The Parquet schema (`ContractCodeOutputParquet`) and converter (`ContractCodeOutput.ToParquet()`) omit `LedgerKeyHashBase64` entirely, so `export_ledger_entry_changes --export-contract-code --write-parquet` silently removes the exact ledger-key encoding from contract-code rows while still claiming success.

**Note**: The original hypothesis also covered contract-data, but `ContractDataOutputParquet` has invalid MAP schema tags (`valuetype=STRING`) that cause parquet-go writer initialization to fail before any rows are written. The contract-data field drop is therefore blocked by an earlier crash and is not a silent data loss in the current codebase. This PoC is narrowed to contract-code only, where the Parquet writer initializes successfully and rows are written with the field silently dropped.

## Trigger

Run `export_ledger_entry_changes --export-contract-code --write-parquet` on any range with Soroban contract-code ledger-entry changes. The JSON rows will contain `ledger_key_hash_base_64`, while the Parquet rows will not have any column for it.

## Target Code

- `internal/transform/contract_code.go:TransformContractCode:28-41` — computes the base64 ledger key for contract-code rows
- `internal/transform/contract_code.go:TransformContractCode:79-99` — assigns `LedgerKeyHashBase64` onto `ContractCodeOutput`
- `internal/transform/schema.go:ContractCodeOutput:539-560` — exposes the contract-code field in the JSON/output schema
- `internal/transform/schema_parquet.go:ContractCodeOutputParquet:283-303` — defines no Parquet column for the field
- `internal/transform/parquet_converter.go:ContractCodeOutput.ToParquet:316-336` — omits the field when building Parquet rows
- `cmd/export_ledger_entry_changes.go:exportTransformedData:340-347` — routes contract-code into the Parquet writer path

## Evidence

The transform does real work to materialize `LedgerKeyHashBase64`: it derives the ledger key from XDR, base64-encodes it, and stores it in the output struct. `internal/transform/contract_code_test.go:112-124` asserts a concrete expected value for that field, showing it is part of the intended row shape. Yet the Parquet schema has nowhere to store it.

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

`TransformContractCode` computes `LedgerKeyHashBase64` by calling `ledgerEntry.LedgerKey()` then `xdr.MarshalBase64(ledgerKey)`, and assigns the result to the output struct. The JSON schema (`ContractCodeOutput`) includes this field and tests assert concrete expected values. However, `ContractCodeOutputParquet` has no corresponding field, and `ToParquet()` silently skips the value. This is an irreversible data drop — `ledger_key_hash` is a one-way SHA256 hash that cannot reconstruct the original XDR ledger key bytes.

### Code Paths Examined

- `internal/transform/contract_code.go:28-41` — computes `ledgerKeyHashBase64` via `xdr.MarshalBase64(ledgerKey)`, confirmed populated
- `internal/transform/contract_code.go:79-99` — assigns `LedgerKeyHashBase64: ledgerKeyHashBase64` in `ContractCodeOutput` struct literal
- `internal/transform/schema.go:560` — `ContractCodeOutput` declares `LedgerKeyHashBase64 string` with JSON tag `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:283-303` — `ContractCodeOutputParquet` has 18 fields; `LedgerKeyHashBase64` is absent
- `internal/transform/parquet_converter.go:316-336` — `ContractCodeOutput.ToParquet()` maps 18 fields; `LedgerKeyHashBase64` not included
- `internal/transform/contract_code_test.go:123` — test asserts concrete base64 value for contract-code

### Findings

The field `LedgerKeyHashBase64` is computed by the transform function, stored in the JSON output struct, and asserted by unit tests. It carries the base64-encoded XDR representation of the ledger key — the full reversible encoding that allows downstream consumers to reconstruct the exact `LedgerKey` XDR object. The companion field `LedgerKeyHash` is a one-way SHA256 hash useful for lookups but incapable of reconstructing the original key. The Parquet schema and `ToParquet()` converter omit this field entirely, meaning Parquet consumers lose the ability to decode the original ledger key.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go`
- **Setup**: Construct a `ContractCodeOutput` with a non-empty `LedgerKeyHashBase64` value
- **Steps**: Write through the actual parquet-go writer (same as production `WriteParquet`), read the file back, check schema columns
- **Assertion**: Assert that the Parquet file has no `LedgerKeyHashBase64` column while the JSON struct retains the field with value

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractCodeParquetDropsLedgerKeyHashBase64"
**Test Language**: Go

### Demonstration

The test constructs a `ContractCodeOutput` with a populated `LedgerKeyHashBase64` value, writes it through the actual parquet-go writer using the production `ContractCodeOutputParquet` schema and `ToParquet()` converter, then reads the Parquet file back and inspects its schema elements. The Parquet file is written successfully (1 row) but contains no `LedgerKeyHashBase64` column, while the companion `LedgerKeyHash` column IS present. This proves the contract-code Parquet export path silently drops the reversible base64 ledger key encoding while retaining only the irreversible SHA256 hash.

### Test Body

```go
package transform

import (
	"os"
	"testing"
	"time"

	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/reader"
	"github.com/xitongsys/parquet-go/writer"
)

// TestContractCodeParquetDropsLedgerKeyHashBase64 demonstrates that the
// contract-code Parquet export path silently drops the LedgerKeyHashBase64
// field. The test writes through the actual parquet-go writer (same path as
// production export_ledger_entry_changes), reads the file back, and shows
// that the column is absent from the Parquet schema.
func TestContractCodeParquetDropsLedgerKeyHashBase64(t *testing.T) {
	// 1. Build a ContractCodeOutput with a non-empty LedgerKeyHashBase64
	cco := ContractCodeOutput{
		ContractCodeHash:    "deadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef",
		ContractCodeExtV:    1,
		LastModifiedLedger:  200,
		LedgerEntryChange:   1,
		Deleted:             false,
		ClosedAt:            time.Date(2024, 6, 15, 12, 0, 0, 0, time.UTC),
		LedgerSequence:      200,
		LedgerKeyHash:       "someSHA256hash",
		NInstructions:       10,
		NFunctions:          5,
		NGlobals:            2,
		NTableEntries:       1,
		NTypes:              3,
		NDataSegments:       1,
		NElemSegments:       0,
		NImports:            4,
		NExports:            2,
		NDataSegmentBytes:   256,
		LedgerKeyHashBase64: "AAAABgAAAADf5PcQ3MN7JEYmumMFPb6C5IVxWxLMB+wNu0g4FCq6cA==",
	}

	if cco.LedgerKeyHashBase64 == "" {
		t.Fatal("test setup error: LedgerKeyHashBase64 is empty")
	}

	// 2. Write through the actual parquet-go production path
	tmpFile, err := os.CreateTemp("", "poc-contract-code-*.parquet")
	if err != nil {
		t.Fatal(err)
	}
	tmpPath := tmpFile.Name()
	tmpFile.Close()
	defer os.Remove(tmpPath)

	pf, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatalf("could not create parquet file writer: %v", err)
	}

	// Same initialization as production WriteParquet in cmd/command_utils.go
	pw, err := writer.NewParquetWriter(pf, new(ContractCodeOutputParquet), 1)
	if err != nil {
		pf.Close()
		t.Fatalf("parquet writer init failed (unexpected): %v", err)
	}

	// Write the row via production ToParquet() converter
	if err := pw.Write(cco.ToParquet()); err != nil {
		t.Fatalf("parquet write failed: %v", err)
	}
	if err := pw.WriteStop(); err != nil {
		t.Fatalf("parquet write stop failed: %v", err)
	}
	pf.Close()

	// 3. Read the Parquet file back and inspect its column schema
	rf, err := local.NewLocalFileReader(tmpPath)
	if err != nil {
		t.Fatalf("could not open parquet file for reading: %v", err)
	}
	defer rf.Close()

	pr, err := reader.NewParquetReader(rf, new(ContractCodeOutputParquet), 1)
	if err != nil {
		t.Fatalf("parquet reader init failed: %v", err)
	}
	defer pr.ReadStop()

	// Collect all column names from the Parquet schema elements.
	// parquet-go uses the Go struct field name (e.g. "LedgerKeyHash") in elements.
	columnNames := make(map[string]bool)
	for _, col := range pr.SchemaHandler.SchemaElements {
		columnNames[col.GetName()] = true
	}

	// 4. Assert that LedgerKeyHashBase64 is absent from the Parquet schema
	if columnNames["LedgerKeyHashBase64"] || columnNames["ledger_key_hash_base_64"] {
		t.Skip("LedgerKeyHashBase64 found in Parquet schema — bug may have been fixed")
	}

	// Confirm that the companion field LedgerKeyHash IS present
	if !columnNames["LedgerKeyHash"] {
		t.Fatal("LedgerKeyHash column missing from Parquet schema — unexpected")
	}

	t.Errorf("BUG CONFIRMED: ContractCodeOutput.LedgerKeyHashBase64 = %q was written through "+
		"the production Parquet path, but the resulting Parquet file has no 'ledger_key_hash_base_64' "+
		"column. The base64-encoded ledger key XDR is silently dropped "+
		"while the one-way SHA256 hash (ledger_key_hash) is retained, causing irreversible data loss "+
		"for Parquet consumers.",
		cco.LedgerKeyHashBase64)
}
```

### Test Output

```
=== RUN   TestContractCodeParquetDropsLedgerKeyHashBase64
    data_integrity_poc_test.go:106: BUG CONFIRMED: ContractCodeOutput.LedgerKeyHashBase64 = "AAAABgAAAADf5PcQ3MN7JEYmumMFPb6C5IVxWxLMB+wNu0g4FCq6cA==" was written through the production Parquet path, but the resulting Parquet file has no 'ledger_key_hash_base_64' column. The base64-encoded ledger key XDR is silently dropped while the one-way SHA256 hash (ledger_key_hash) is retained, causing irreversible data loss for Parquet consumers.
--- FAIL: TestContractCodeParquetDropsLedgerKeyHashBase64 (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.877s
```
