# 022: Contract-code Parquet drops `ledger_key_hash_base_64`

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`TransformContractCode()` computes a reversible base64 encoding of the contract-code ledger key and exposes it on `ContractCodeOutput.LedgerKeyHashBase64`, but the Parquet schema and converter omit that field. As a result, `export_ledger_entry_changes --export-contract-code --write-parquet` writes successful Parquet rows that keep only the irreversible `ledger_key_hash` and silently discard the original ledger-key encoding.

## Root Cause

The contract-code transform populates `LedgerKeyHashBase64` in the JSON/output struct, but `ContractCodeOutputParquet` has no matching column and `ContractCodeOutput.ToParquet()` never copies the value. The shared `WriteParquet()` path writes only the reduced Parquet struct, so every contract-code row loses the field during Parquet serialization.

## Reproduction

Run `export_ledger_entry_changes --export-contract-code --write-parquet` on any ledger range containing a `LedgerEntryTypeContractCode` change. The production path computes `ledger_key_hash_base_64`, emits it in JSON-shaped `ContractCodeOutput`, then writes Parquet rows whose schema has no column for that field.

## Affected Code

- `internal/transform/contract_code.go:TransformContractCode:12-100` — computes `ledgerKeyHashBase64` from `ledgerEntry.LedgerKey()` and assigns it to `ContractCodeOutput`
- `internal/transform/schema.go:ContractCodeOutput:539-560` — exported row schema includes `LedgerKeyHashBase64`
- `internal/transform/schema_parquet.go:ContractCodeOutputParquet:283-303` — Parquet row schema omits the field
- `internal/transform/parquet_converter.go:ContractCodeOutput.ToParquet:316-336` — converter drops the field while copying the rest
- `cmd/command_utils.go:WriteParquet:162-179` — production Parquet writer persists only the reduced Parquet struct

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractCodeParquetDropsLedgerKeyHashBase64`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./internal/transform/... -run '^TestContractCodeParquetDropsLedgerKeyHashBase64$' -v`.

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

	tmpFile, err := os.CreateTemp("", "poc-contract-code-*.parquet")
	if err != nil {
		t.Fatal(err)
	}
	tmpPath := tmpFile.Name()
	if err := tmpFile.Close(); err != nil {
		t.Fatal(err)
	}
	defer os.Remove(tmpPath)

	pf, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatalf("could not create parquet file writer: %v", err)
	}

	pw, err := writer.NewParquetWriter(pf, new(ContractCodeOutputParquet), 1)
	if err != nil {
		_ = pf.Close()
		t.Fatalf("parquet writer init failed (unexpected): %v", err)
	}

	if err := pw.Write(cco.ToParquet()); err != nil {
		t.Fatalf("parquet write failed: %v", err)
	}
	if err := pw.WriteStop(); err != nil {
		t.Fatalf("parquet write stop failed: %v", err)
	}
	if err := pf.Close(); err != nil {
		t.Fatalf("parquet file close failed: %v", err)
	}

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

	columnNames := make(map[string]bool)
	for _, col := range pr.SchemaHandler.SchemaElements {
		columnNames[col.GetName()] = true
	}

	if columnNames["LedgerKeyHashBase64"] || columnNames["ledger_key_hash_base_64"] {
		t.Skip("LedgerKeyHashBase64 found in Parquet schema - bug may have been fixed")
	}

	if !columnNames["LedgerKeyHash"] {
		t.Fatal("LedgerKeyHash column missing from Parquet schema - unexpected")
	}

	t.Errorf("BUG CONFIRMED: ContractCodeOutput.LedgerKeyHashBase64 = %q was written through "+
		"the production Parquet path, but the resulting Parquet file has no 'ledger_key_hash_base_64' "+
		"column. The base64-encoded ledger key XDR is silently dropped "+
		"while the one-way SHA256 hash (ledger_key_hash) is retained, causing irreversible data loss "+
		"for Parquet consumers.",
		cco.LedgerKeyHashBase64)
}
```

## Expected vs Actual Behavior

- **Expected**: Contract-code Parquet rows preserve the same `ledger_key_hash_base_64` value that the transform computes and exposes on `ContractCodeOutput`.
- **Actual**: Contract-code Parquet rows have no `ledger_key_hash_base_64` column at all, so the reversible ledger-key encoding is lost while export still reports success.

## Adversarial Review

1. Exercises claimed bug: **YES** — the PoC calls the real `ContractCodeOutput.ToParquet()` conversion and writes through the same `parquet-go` writer used by `WriteParquet()`.
2. Realistic preconditions: **YES** — any normal `--export-contract-code --write-parquet` run that encounters a contract-code change hits this path.
3. Bug vs by-design: **BUG** — unlike intentionally excluded large payloads, this field is actively computed, stored on the exported row struct, and asserted by existing unit tests with no documentation of a JSON/Parquet schema split.
4. Final severity: **High** — this is silent structural corruption of a non-financial field for every affected Parquet row.
5. In scope: **YES** — it is a concrete production code path producing wrong exported data.
6. Test correctness: **CORRECT** — the test proves the specific value exists before serialization and that the resulting Parquet schema lacks any destination column for it.
7. Alternative explanations: **NONE** — there is no earlier crash on the contract-code Parquet path and no documented rationale for dropping only this field.
8. Novelty: **NOT ASSESSED HERE** — duplicate handling is managed outside this final review.

## Suggested Fix

Add `LedgerKeyHashBase64 string` to `ContractCodeOutputParquet` with the appropriate Parquet tag, copy it in `ContractCodeOutput.ToParquet()`, and keep JSON and Parquet row shapes consistent for contract-code exports.
