# 012: Transaction Parquet fabricates absent optional fields

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` intentionally leaves several muxed-account and fee-bump-only fields unset on classic transactions, and the JSON schema preserves that absence with `omitempty`. The Parquet path destroys that distinction by forcing those fields into required `string` and `int64` columns, so classic rows export `""` / `0` instead of null or absent values.

This is a real structural corruption bug in normal export flow. `export_transactions --write-parquet` collects `TransactionOutput` rows and writes `TransactionOutputParquet` via `ToParquet()`, so downstream Parquet consumers see fabricated values for `account_muxed`, `fee_account`, `fee_account_muxed`, `inner_transaction_hash`, and `new_max_fee`.

## Root Cause

`TransformTransaction()` builds `TransactionOutput` with Go zero values and only populates `AccountMuxed` inside the muxed-source branch and `FeeAccount`, `FeeAccountMuxed`, `InnerTransactionHash`, and `NewMaxFee` inside the fee-bump branch. The JSON schema marks those fields `omitempty`, but `TransactionOutputParquet` narrows them to required fields and `TransactionOutput.ToParquet()` copies the zero values directly, converting absence into concrete data.

## Reproduction

Run `export_transactions --write-parquet` on any classic non-fee-bump transaction with a non-muxed source account. At the transform/JSON layer, the five fields above are absent; after Parquet conversion, the same row contains `account_muxed=""`, `fee_account=""`, `fee_account_muxed=""`, `inner_transaction_hash=""`, and `new_max_fee=0`.

## Affected Code

- `internal/transform/transaction.go:233-301` — constructs `TransactionOutput` and only sets the five fields inside muxed-account / fee-bump conditionals.
- `internal/transform/schema.go:45,60-63` — JSON schema models those fields as omittable.
- `internal/transform/schema_parquet.go:36,51-54` — Parquet schema makes those fields required `string` / `int64` columns.
- `internal/transform/parquet_converter.go:59-82` — Parquet conversion writes zero values directly into the required columns.
- `cmd/export_transactions.go:33-66` — production export path appends `TransactionOutput` rows and writes them as `TransactionOutputParquet`.
- `cmd/command_utils.go:162-176` — Parquet writer emits the result of each row's `ToParquet()` method.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTransactionParquetFabricatesOptionalFields`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestTransactionParquetFabricatesOptionalFields -v`
  4. Observe the JSON layer omit the fields while the Parquet row contains `""` / `0`

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"
)

func TestTransactionParquetFabricatesOptionalFields(t *testing.T) {
	transactions, headers, err := makeTransactionTestInput()
	if err != nil {
		t.Fatalf("makeTransactionTestInput() failed: %v", err)
	}

	classicTx := transactions[0]
	classicHeader := headers[0]
	if classicTx.Envelope.IsFeeBump() {
		t.Fatal("expected first fixture transaction to be classic")
	}

	output, err := TransformTransaction(classicTx, classicHeader)
	if err != nil {
		t.Fatalf("TransformTransaction() failed: %v", err)
	}

	jsonBytes, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("json.Marshal() failed: %v", err)
	}

	var jsonMap map[string]any
	if err := json.Unmarshal(jsonBytes, &jsonMap); err != nil {
		t.Fatalf("json.Unmarshal() failed: %v", err)
	}

	for _, field := range []string{
		"account_muxed",
		"fee_account",
		"fee_account_muxed",
		"inner_transaction_hash",
		"new_max_fee",
	} {
		if _, ok := jsonMap[field]; ok {
			t.Fatalf("expected JSON to omit %q for classic transaction", field)
		}
	}

	parquet := output.ToParquet().(TransactionOutputParquet)
	if parquet.AccountMuxed != "" {
		t.Fatalf("expected parquet account_muxed to be empty string, got %q", parquet.AccountMuxed)
	}
	if parquet.FeeAccount != "" {
		t.Fatalf("expected parquet fee_account to be empty string, got %q", parquet.FeeAccount)
	}
	if parquet.FeeAccountMuxed != "" {
		t.Fatalf("expected parquet fee_account_muxed to be empty string, got %q", parquet.FeeAccountMuxed)
	}
	if parquet.InnerTransactionHash != "" {
		t.Fatalf("expected parquet inner_transaction_hash to be empty string, got %q", parquet.InnerTransactionHash)
	}
	if parquet.NewMaxFee != 0 {
		t.Fatalf("expected parquet new_max_fee to be 0, got %d", parquet.NewMaxFee)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Classic transactions should preserve the absence of fee-bump and muxed-only metadata in Parquet, so those columns remain null or otherwise distinguishable from concrete values.
- **Actual**: The Parquet export fabricates concrete `""` and `0` values for all five absent fields.

## Adversarial Review

1. Exercises claimed bug: YES — the validated PoC uses the package's real classic transaction fixture, runs `TransformTransaction()`, marshals the real output to JSON, then runs the production `ToParquet()` converter.
2. Realistic preconditions: YES — classic non-fee-bump transactions with unmuxed source accounts are standard Stellar traffic and match the first built-in transaction fixture.
3. Bug vs by-design: BUG — the ETL already chose absent semantics in `TransactionOutput` via conditional population plus `omitempty`; only the Parquet layer discards that information.
4. Final severity: High — this is silent structural corruption of non-financial transaction metadata that can break null-sensitive analytics and fee-bump attribution logic.
5. In scope: YES — it is a concrete transform/export path that produces plausible but wrong Parquet data during normal operation.
6. Test correctness: CORRECT — the test proves the fields are absent in the real JSON representation before conversion and present as fabricated values only after Parquet conversion.
7. Alternative explanations: NONE — while `""` is not a valid account or hash, it is still non-null data and changes query semantics; `new_max_fee=0` likewise fabricates presence for a field that should be absent.
8. Novelty: NOVEL

## Suggested Fix

Make the affected Parquet fields nullable, such as `*string` / `*int64` with optional Parquet tags, and emit `nil` when the corresponding transaction metadata is absent. Audit the rest of `TransactionOutputParquet` for the same optional-in-JSON to required-in-Parquet pattern so other conditionally populated fields do not flatten to sentinel values.
