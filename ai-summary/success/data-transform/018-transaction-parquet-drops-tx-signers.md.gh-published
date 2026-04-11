# 018: Transaction Parquet export drops `tx_signers`

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` populates `TransactionOutput.TxSigners`, and the JSON transaction schema exposes that signer array, but the Parquet transaction schema never added a matching column. As a result, `export_transactions --write-parquet` silently omits signer-set data from every exported Parquet row even when the source `TransactionOutput` is populated.

This is a real structural corruption bug in the normal export flow. The production export path appends transformed `TransactionOutput` rows, converts them through `TransactionOutput.ToParquet()`, and builds the file schema from `TransactionOutputParquet`, so downstream Parquet consumers have no way to recover `tx_signers`.

## Root Cause

The transaction transform layer and JSON schema both treat `tx_signers` as part of the exported transaction contract, but the Parquet surface drifted behind them. `TransactionOutputParquet` has no `TxSigners` field, and `TransactionOutput.ToParquet()` therefore has nowhere to copy signer data before `WriteParquet()` emits the row.

## Reproduction

Run `export_transactions --write-parquet` on any normal Stellar transaction row with signatures. `TransformTransaction()` produces a `TransactionOutput` whose `TxSigners` slice is populated, but the converted `TransactionOutputParquet` row has no `TxSigners` field and the generated Parquet schema has no `tx_signers` column, so the final file drops the signer list entirely.

## Affected Code

- `internal/transform/transaction.go:227-300` — populates `TransactionOutput.TxSigners` from transaction or fee-bump signatures.
- `internal/transform/schema.go:41-84` — JSON/export schema includes `TxSigners []string`.
- `internal/transform/schema_parquet.go:32-74` — Parquet transaction schema omits any `TxSigners` field.
- `internal/transform/parquet_converter.go:59-102` — `TransactionOutput.ToParquet()` has no mapping for `TxSigners`.
- `cmd/export_transactions.go:33-66` — production export path writes transformed transactions as `TransactionOutputParquet`.
- `cmd/command_utils.go:162-176` — Parquet writer emits each row from `record.ToParquet()` and derives the file schema from the supplied Parquet struct.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestParquetDropsTxSigners`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestParquetDropsTxSigners -v`
  4. Observe the failure showing that the converted row type and generated Parquet schema both omit `TxSigners`

### Test Body

```go
package transform

import (
	"path/filepath"
	"reflect"
	"strings"
	"testing"

	"github.com/lib/pq"
	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/writer"
)

func TestParquetDropsTxSigners(t *testing.T) {
	txOutput := TransactionOutput{
		TransactionHash: "abc123",
		TxSigners:       []string{"GABCDEF", "GHIJKLM"},
		ExtraSigners:    pq.StringArray{"TEXTRA"},
	}
	if len(txOutput.TxSigners) != 2 {
		t.Fatalf("setup failed: expected 2 tx_signers, got %d", len(txOutput.TxSigners))
	}

	parquetRow := txOutput.ToParquet()
	rowValue := reflect.ValueOf(parquetRow)
	hasTxSignersField := rowValue.FieldByName("TxSigners").IsValid()

	parquetPath := filepath.Join(t.TempDir(), "transactions.parquet")
	parquetFile, err := local.NewLocalFileWriter(parquetPath)
	if err != nil {
		t.Fatalf("create parquet file writer: %v", err)
	}
	var parquetWriter *writer.ParquetWriter
	t.Cleanup(func() {
		if parquetWriter != nil {
			if err := parquetWriter.WriteStop(); err != nil {
				t.Errorf("stop parquet writer: %v", err)
			}
		}
		if err := parquetFile.Close(); err != nil {
			t.Errorf("close parquet file: %v", err)
		}
	})

	parquetWriter, err = writer.NewParquetWriter(parquetFile, new(TransactionOutputParquet), 1)
	if err != nil {
		t.Fatalf("create parquet schema: %v", err)
	}
	if err := parquetWriter.Write(parquetRow); err != nil {
		t.Fatalf("write parquet row: %v", err)
	}

	columns := parquetWriter.SchemaHandler.ValueColumns
	columnsText := strings.Join(columns, "\n")
	if !strings.Contains(columnsText, "ExtraSigners") {
		t.Fatalf("sanity check failed: expected schema to include ExtraSigners, got columns %v", columns)
	}
	if !hasTxSignersField || !strings.Contains(columnsText, "TxSigners") {
		t.Fatalf(
			"expected tx_signers to survive parquet conversion for populated TransactionOutput; row type=%T hasTxSignersField=%t columns=%v input=%v",
			parquetRow,
			hasTxSignersField,
			columns,
			txOutput.TxSigners,
		)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: The transaction Parquet schema should include a `tx_signers` column, and `TransactionOutput.ToParquet()` should preserve the populated signer slice.
- **Actual**: The converted row type has no `TxSigners` field and the generated Parquet schema has no `tx_signers` column, so signer data is dropped from every Parquet export row.

## Adversarial Review

1. Exercises claimed bug: YES — the validated PoC populates a real `TransactionOutput`, runs the production `ToParquet()` converter, and then instantiates the real `parquet-go` writer schema used by exports.
2. Realistic preconditions: YES — every Stellar transaction is signed, and `TransformTransaction()` already computes `TxSigners` from the envelope signatures during normal export flow.
3. Bug vs by-design: BUG — the transform layer and JSON schema explicitly expose `tx_signers`, and the same Parquet struct already supports a sibling repeated signer-like field (`ExtraSigners`), so the omission is schema drift rather than an intentional Parquet limitation.
4. Final severity: High — this is silent structural corruption of transaction metadata in a primary export format, preventing signer-set analytics and reconciliation from using Parquet output.
5. In scope: YES — it is a concrete transform/export path that silently emits incomplete data during normal operation.
6. Test correctness: CORRECT — the PoC proves the omission in both the converted row type and the writer-generated Parquet schema, while also checking that a comparable repeated field (`ExtraSigners`) is present.
7. Alternative explanations: NONE — the absence is not explained by unsupported list fields because the same schema already emits `ExtraSigners`.
8. Novelty: NOVEL

## Suggested Fix

Add `TxSigners []string` to `TransactionOutputParquet` using the same repeated-string Parquet tagging pattern already used for `ExtraSigners`, then update `TransactionOutput.ToParquet()` to copy `to.TxSigners` into that field. Afterward, audit the rest of the transaction Parquet schema for JSON/Parquet drift so newly added JSON fields do not silently disappear from Parquet exports.
