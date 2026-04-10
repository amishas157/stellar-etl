# 007: contract-code Parquet drops populated `ledger_key_hash_base_64`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformContractCode()` computes and exports both a hex `ledger_key_hash` and a base64 `ledger_key_hash_base_64` for each contract-code row. The Parquet path silently drops the base64 form because `ContractCodeOutputParquet` has no corresponding field and `ContractCodeOutput.ToParquet()` never copies it, so Parquet consumers lose a populated identifier that the JSON export preserves.

## Root Cause

The contract-code transform stores `LedgerKeyHashBase64` on `ContractCodeOutput`, but the Parquet schema in `schema_parquet.go` stops at `NDataSegmentBytes`. The converter in `parquet_converter.go` mirrors that incomplete schema, so every `--write-parquet` contract-code export discards the base64 ledger-key encoding.

## Reproduction

During normal operation, `export_ledger_entry_changes --export-contract-code --write-parquet` transforms each `LedgerEntryTypeContractCode` change into `ContractCodeOutput` and then appends it to the Parquet batch as `ContractCodeOutputParquet`. For any contract-code entry, the JSON row contains a non-empty `ledger_key_hash_base_64`, but the Parquet row type lacks that column entirely, so the value cannot survive conversion.

## Affected Code

- `internal/transform/contract_code.go:12-100` — computes `ledgerKeyHashBase64` from the ledger key and stores it on `ContractCodeOutput`
- `internal/transform/schema.go:539-560` — JSON schema exposes `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:283-303` — Parquet schema omits `LedgerKeyHashBase64`
- `internal/transform/parquet_converter.go:316-336` — `ToParquet()` omits the populated field
- `cmd/export_ledger_entry_changes.go:232-243` — live export path produces `ContractCodeOutput`
- `cmd/export_ledger_entry_changes.go:340-342` — Parquet export selects `ContractCodeOutputParquet`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractCodeParquetDropsLedgerKeyHashBase64`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestContractCodeParquetDropsLedgerKeyHashBase64 -v`
  4. Observe that the test fails because the Parquet row type has no `LedgerKeyHashBase64` field even though the transformed JSON row populated it.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestContractCodeParquetDropsLedgerKeyHashBase64(t *testing.T) {
	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	contractCode, err := TransformContractCode(makeContractCodeTestInput()[0], header)
	if err != nil {
		t.Fatalf("TransformContractCode() error = %v", err)
	}

	if contractCode.LedgerKeyHashBase64 == "" {
		t.Fatal("precondition failed: TransformContractCode() should populate LedgerKeyHashBase64")
	}

	parquetValue, ok := contractCode.ToParquet().(ContractCodeOutputParquet)
	if !ok {
		t.Fatalf("ToParquet() returned %T, want ContractCodeOutputParquet", contractCode.ToParquet())
	}

	if parquetValue.LedgerKeyHash != contractCode.LedgerKeyHash {
		t.Fatalf("precondition failed: parquet preserved ledger_key_hash=%q, want %q", parquetValue.LedgerKeyHash, contractCode.LedgerKeyHash)
	}

	if _, hasField := reflect.TypeOf(parquetValue).FieldByName("LedgerKeyHashBase64"); !hasField {
		t.Fatalf("ledger_key_hash_base_64 lost in parquet conversion: JSON output had %q, but ContractCodeOutputParquet has no LedgerKeyHashBase64 field", contractCode.LedgerKeyHashBase64)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Contract-code Parquet rows should preserve the same non-empty `ledger_key_hash_base_64` value that `TransformContractCode()` computes and the JSON export emits.
- **Actual**: `ContractCodeOutputParquet` has no `ledger_key_hash_base_64` field, so `ToParquet()` drops the populated value from every Parquet row.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs the real `TransformContractCode()` path on the repository's existing contract-code fixture, proves the JSON row has a populated base64 ledger key, and fails only because `ToParquet()` converts into a struct with no destination field.
2. Realistic preconditions: YES — any `LedgerEntryTypeContractCode` row exported through `export_ledger_entry_changes --export-contract-code --write-parquet` follows this exact path.
3. Bug vs by-design: BUG — the JSON schema and existing contract-code tests both treat `ledger_key_hash_base_64` as part of the exported row, and there is no code or documentation indicating Parquet should intentionally discard it.
4. Final severity: High — this is structural export corruption that removes a populated identifier from Parquet output, but it does not directly alter monetary values.
5. In scope: YES — production export logic silently emits incomplete Parquet data.
6. Test correctness: CORRECT — the test checks the real transformed source value before conversion and then inspects the concrete Parquet row type to show the field cannot be preserved.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Add `LedgerKeyHashBase64` to `ContractCodeOutputParquet` and populate it in `ContractCodeOutput.ToParquet()` so the Parquet export preserves the same base64 ledger-key encoding as the JSON export.
