# 014: Contract-code Parquet export drops ledger-key base64

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformContractCode()` computes and exports both `ledger_key_hash` and `ledger_key_hash_base_64` for contract-code rows, and the JSON path preserves both values. The Parquet path silently drops `ledger_key_hash_base_64` because neither `ContractCodeOutputParquet` nor `ContractCodeOutput.ToParquet()` has a destination for it, so every `export_ledger_entry_changes --write-parquet` contract-code row loses information that JSON still carries.

## Root Cause

`internal/transform/contract_code.go` populates `LedgerKeyHashBase64` on the JSON/output struct, but the parallel Parquet schema and converter never added the corresponding string field. `cmd/export_ledger_entry_changes.go` still routes `ContractCodeOutput` rows into `WriteParquet()`, which serializes `record.ToParquet()` directly, so the omission reaches emitted Parquet artifacts without any error.

## Reproduction

On any ledger range containing `CONTRACT_CODE` changes, `export_ledger_entry_changes --write-parquet` will transform each row with a non-empty base64-encoded ledger key and then serialize it through `ContractCodeOutput.ToParquet()`. Because the Parquet struct lacks `ledger_key_hash_base_64`, downstream Parquet consumers see only the irreversible hash and cannot recover the serialized ledger-key payload that the JSON export for the same row includes.

## Affected Code

- `internal/transform/contract_code.go:28-41` — computes `ledgerKeyHashBase64` from the live ledger key
- `internal/transform/contract_code.go:79-99` — assigns `LedgerKeyHashBase64` onto `ContractCodeOutput`
- `internal/transform/schema.go:540-560` — JSON/output schema exposes `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:284-303` — Parquet schema omits any `ledger_key_hash_base_64` column
- `internal/transform/parquet_converter.go:316-336` — `ToParquet()` drops the populated base64 value
- `cmd/export_ledger_entry_changes.go:340-343` — contract-code rows are routed into the Parquet export path
- `cmd/command_utils.go:162-180` — `WriteParquet()` serializes the truncated Parquet struct

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractCodeParquetDropsLedgerKeyHashBase64`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

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

func TestContractCodeParquetDropsLedgerKeyHashBase64(t *testing.T) {
	input := ContractCodeOutput{
		ContractCodeHash:    "deadbeef",
		LedgerKeyHash:       "abc123hash",
		LedgerKeyHashBase64: "AAAABwAAAAEAAAAAAAAAAgAAAA==",
		LedgerSequence:      200,
		ClosedAt:            time.Unix(0, 0).UTC(),
	}

	parquetResult := input.ToParquet()

	if input.LedgerKeyHashBase64 == "" {
		t.Fatal("precondition failed: input LedgerKeyHashBase64 must be populated")
	}

	jsonType := reflect.TypeOf(input)
	if _, ok := jsonType.FieldByName("LedgerKeyHashBase64"); !ok {
		t.Fatal("expected ContractCodeOutput to expose LedgerKeyHashBase64")
	}

	parquetType := reflect.TypeOf(parquetResult)
	if _, ok := parquetType.FieldByName("LedgerKeyHashBase64"); ok {
		t.Fatal("expected ContractCodeOutputParquet to omit LedgerKeyHashBase64")
	}

	tmpFile, err := os.CreateTemp("", "contract_code_*.parquet")
	if err != nil {
		t.Fatalf("create temp file: %v", err)
	}
	tmpPath := tmpFile.Name()
	if err := tmpFile.Close(); err != nil {
		t.Fatalf("close temp file: %v", err)
	}
	t.Cleanup(func() {
		_ = os.Remove(tmpPath)
	})

	pf, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatalf("create parquet writer: %v", err)
	}
	pw, err := writer.NewParquetWriter(pf, new(ContractCodeOutputParquet), 1)
	if err != nil {
		_ = pf.Close()
		t.Fatalf("init parquet writer: %v", err)
	}

	if err := pw.Write(parquetResult); err != nil {
		_ = pw.WriteStop()
		_ = pf.Close()
		t.Fatalf("write parquet row: %v", err)
	}
	if err := pw.WriteStop(); err != nil {
		_ = pf.Close()
		t.Fatalf("stop parquet writer: %v", err)
	}
	if err := pf.Close(); err != nil {
		t.Fatalf("close parquet file: %v", err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Contract-code Parquet output preserves the same populated `ledger_key_hash_base_64` string that JSON output emits for the same row.
- **Actual**: Contract-code Parquet output has no `ledger_key_hash_base_64` destination at all, so the value is silently discarded from every row.

## Adversarial Review

1. Exercises claimed bug: YES — the test uses the real `ContractCodeOutput.ToParquet()` conversion and the same Parquet writer shape that production export uses.
2. Realistic preconditions: YES — any normal `export_ledger_entry_changes --write-parquet` run over ledgers with contract-code changes reaches this path.
3. Bug vs by-design: BUG — JSON already exports the field, the omitted Parquet field is a plain string that Parquet supports, and there is no skip comment or alternative representation for it.
4. Final severity: High — this is silent structural data loss in a live export artifact, not a direct financial miscalculation.
5. In scope: YES — the production code emits plausible but incomplete Parquet rows without crashing or logging an error.
6. Test correctness: CORRECT — it proves the source struct contains a populated value, the production Parquet struct lacks any receiving field, and the writer still accepts and writes the truncated row.
7. Alternative explanations: NONE — the missing field is not explained by unsupported types or dead code; it is a string omitted from a live serializer.
8. Novelty: NOT ASSESSED HERE — duplicate handling is orchestrator-owned.

## Suggested Fix

Add `LedgerKeyHashBase64 string` with the `parquet:"name=ledger_key_hash_base_64, ..."` tag to `ContractCodeOutputParquet`, and populate it from `cco.LedgerKeyHashBase64` in `ContractCodeOutput.ToParquet()`. A follow-up parity pass should check the neighboring `ContractData` Parquet schema for the same omission pattern.
