# 006: contract-data Parquet drops populated `ledger_key_hash_base_64`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformContractData()` computes and exports both a hex `ledger_key_hash` and a base64 `ledger_key_hash_base_64` for each non-nonce contract-data row. The Parquet path silently drops the base64 form because `ContractDataOutputParquet` has no corresponding field and `ContractDataOutput.ToParquet()` never copies it, so Parquet consumers lose a populated identifier that the JSON export preserves.

## Root Cause

The contract-data transform stores `LedgerKeyHashBase64` on `ContractDataOutput`, but the Parquet schema in `schema_parquet.go` stops at `ContractDataXDR`. The converter in `parquet_converter.go` mirrors that incomplete schema, so every `--write-parquet` contract-data export discards the base64 ledger-key encoding.

## Reproduction

During normal operation, `export_ledger_entry_changes --export-contract-data --write-parquet` transforms each `LedgerEntryTypeContractData` change into `ContractDataOutput` and then appends it to the Parquet batch as `ContractDataOutputParquet`. For any non-nonce contract-data entry, the JSON row contains a non-empty `ledger_key_hash_base_64`, but the Parquet row type lacks that column entirely, so the value cannot survive conversion.

## Affected Code

- `internal/transform/contract_data.go:65-78` — computes `ledgerKeyHashBase64` from the ledger key XDR
- `internal/transform/contract_data.go:135-156` — writes `LedgerKeyHashBase64` into `ContractDataOutput`
- `internal/transform/schema.go:530-536` — JSON schema exposes `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:260-281` — Parquet schema omits `LedgerKeyHashBase64`
- `internal/transform/parquet_converter.go:292-313` — `ToParquet()` omits the populated field
- `cmd/export_ledger_entry_changes.go:212-230` — live export path produces `ContractDataOutput`
- `cmd/export_ledger_entry_changes.go:321-347` — Parquet export selects `ContractDataOutputParquet`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractDataParquetDropsLedgerKeyHashBase64`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestContractDataParquetDropsLedgerKeyHashBase64 -v`
  4. Observe that the test fails because the Parquet row type has no `LedgerKeyHashBase64` field even though the transformed JSON row populated it.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestContractDataParquetDropsLedgerKeyHashBase64(t *testing.T) {
	inputs := makeContractDataTestInput()
	if len(inputs) == 0 {
		t.Fatal("expected at least one contract-data test fixture")
	}

	header := makeContractDataPoCHeader()
	transformer := NewTransformContractDataStruct(MockAssetFromContractData, MockContractBalanceFromContractData)

	output, err, ok := transformer.TransformContractData(inputs[0], "unit test", header)
	if err != nil {
		t.Fatalf("TransformContractData() error = %v", err)
	}
	if !ok {
		t.Fatal("expected TransformContractData() to emit a contract-data row")
	}
	if output.LedgerKeyHashBase64 == "" {
		t.Fatal("expected TransformContractData() to populate LedgerKeyHashBase64")
	}

	parquetOutput, ok := output.ToParquet().(ContractDataOutputParquet)
	if !ok {
		t.Fatal("ToParquet() did not return ContractDataOutputParquet")
	}

	parquetType := reflect.TypeOf(parquetOutput)
	if _, exists := parquetType.FieldByName("LedgerKeyHashBase64"); !exists {
		t.Fatalf("ContractDataOutputParquet is missing LedgerKeyHashBase64, so parquet conversion drops populated JSON data %q", output.LedgerKeyHashBase64)
	}

	got := reflect.ValueOf(parquetOutput).FieldByName("LedgerKeyHashBase64").String()
	if got != output.LedgerKeyHashBase64 {
		t.Fatalf("LedgerKeyHashBase64 corrupted during parquet conversion: got %q, want %q", got, output.LedgerKeyHashBase64)
	}
}

func makeContractDataPoCHeader() xdr.LedgerHeaderHistoryEntry {
	return xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Contract-data Parquet rows should preserve the same non-empty `ledger_key_hash_base_64` value that `TransformContractData()` computes and the JSON export emits.
- **Actual**: `ContractDataOutputParquet` has no `ledger_key_hash_base_64` field, so `ToParquet()` drops the populated value from every Parquet row.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC starts with the repository's contract-data fixture, runs `TransformContractData()`, proves the source row has a populated base64 ledger key, and fails only because `ToParquet()` converts into a struct with no destination field.
2. Realistic preconditions: YES — any non-nonce `LedgerEntryTypeContractData` row exported through `export_ledger_entry_changes --export-contract-data --write-parquet` follows this exact path.
3. Bug vs by-design: BUG — the JSON schema and existing contract-data tests both treat `ledger_key_hash_base_64` as part of the exported row, and there is no code or documentation indicating Parquet should intentionally discard it.
4. Final severity: High — this is structural export corruption that removes a populated identifier from Parquet output, but it does not directly alter monetary values.
5. In scope: YES — production export logic silently emits incomplete Parquet data.
6. Test correctness: CORRECT — the test checks the real transformed source value before conversion and then inspects the concrete Parquet row type to show the field cannot be preserved.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Add `LedgerKeyHashBase64` to `ContractDataOutputParquet` and populate it in `ContractDataOutput.ToParquet()` so the Parquet export preserves the same base64 ledger-key encoding as the JSON export.
