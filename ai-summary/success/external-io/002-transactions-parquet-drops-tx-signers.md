# 002: Transaction Parquet drops `tx_signers`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` populates `tx_signers` for both classic and fee-bump transactions, and the JSON export preserves that field. The Parquet export path drops it entirely because the Parquet schema and converter never carry `TxSigners` forward, so every `export_transactions --write-parquet` run silently loses signer data that exists in the source row.

## Root Cause

`TransactionOutputParquet` has no `TxSigners` field, and `TransactionOutput.ToParquet()` correspondingly has no assignment for it. `cmd/export_transactions.go` writes those Parquet rows directly, so the emitted file schema has no `tx_signers` column and cannot represent the populated source value.

## Reproduction

Any normal `export_transactions --write-parquet` run that includes a signed transaction reaches this path. `TransformTransaction()` returns a `TransactionOutput` with non-empty `TxSigners`, but `ToParquet()` converts it into a `TransactionOutputParquet` that has no place to store those values before the writer serializes the row.

## Affected Code

- `internal/transform/transaction.go:227-300` — populates `TxSigners` on both classic and fee-bump paths
- `internal/transform/schema.go:41-84` — exposes `TxSigners []string` in the JSON transaction schema
- `internal/transform/schema_parquet.go:31-74` — defines `TransactionOutputParquet` without any `TxSigners` field
- `internal/transform/parquet_converter.go:59-103` — converts transactions to Parquet rows without copying signer data
- `cmd/export_transactions.go:63-65` — writes the lossy Parquet rows when `--write-parquet` is enabled

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestParquetDropsTxSigners`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"encoding/json"
	"path/filepath"
	"testing"

	local "github.com/xitongsys/parquet-go-source/local"
	parquetreader "github.com/xitongsys/parquet-go/reader"
	parquetwriter "github.com/xitongsys/parquet-go/writer"
)

func TestParquetDropsTxSigners(t *testing.T) {
	transactions, historyHeaders, err := makeTransactionTestInput()
	if err != nil {
		t.Fatalf("makeTransactionTestInput() error = %v", err)
	}

	transformed, err := TransformTransaction(transactions[2], historyHeaders[2])
	if err != nil {
		t.Fatalf("TransformTransaction() error = %v", err)
	}

	if len(transformed.TxSigners) == 0 {
		t.Fatal("test fixture is invalid: TransformTransaction() returned no tx_signers")
	}
	if len(transformed.ExtraSigners) == 0 {
		t.Fatal("test fixture is invalid: TransformTransaction() returned no extra_signers")
	}

	jsonBytes, err := json.Marshal(transformed)
	if err != nil {
		t.Fatalf("json.Marshal(TransactionOutput) error = %v", err)
	}

	var jsonOutput map[string]interface{}
	if err := json.Unmarshal(jsonBytes, &jsonOutput); err != nil {
		t.Fatalf("json.Unmarshal(TransactionOutput) error = %v", err)
	}

	if _, ok := jsonOutput["tx_signers"]; !ok {
		t.Fatal("JSON export is missing tx_signers")
	}
	if _, ok := jsonOutput["extra_signers"]; !ok {
		t.Fatal("JSON export is missing extra_signers")
	}

	parquetPath := filepath.Join(t.TempDir(), "transactions.parquet")
	parquetFile, err := local.NewLocalFileWriter(parquetPath)
	if err != nil {
		t.Fatalf("NewLocalFileWriter() error = %v", err)
	}

	writer, err := parquetwriter.NewParquetWriter(parquetFile, new(TransactionOutputParquet), 1)
	if err != nil {
		_ = parquetFile.Close()
		t.Fatalf("NewParquetWriter() error = %v", err)
	}

	if err := writer.Write(transformed.ToParquet()); err != nil {
		_ = writer.WriteStop()
		_ = parquetFile.Close()
		t.Fatalf("writer.Write() error = %v", err)
	}
	if err := writer.WriteStop(); err != nil {
		_ = parquetFile.Close()
		t.Fatalf("writer.WriteStop() error = %v", err)
	}
	if err := parquetFile.Close(); err != nil {
		t.Fatalf("parquetFile.Close() error = %v", err)
	}

	parquetReaderFile, err := local.NewLocalFileReader(parquetPath)
	if err != nil {
		t.Fatalf("NewLocalFileReader() error = %v", err)
	}

	parquetReader, err := parquetreader.NewParquetReader(parquetReaderFile, new(TransactionOutputParquet), 1)
	if err != nil {
		_ = parquetReaderFile.Close()
		t.Fatalf("NewParquetReader() error = %v", err)
	}
	defer parquetReader.ReadStop()

	readBack := make([]TransactionOutputParquet, 1)
	if err := parquetReader.Read(&readBack); err != nil {
		t.Fatalf("parquetReader.Read() error = %v", err)
	}
	if len(readBack) != 1 {
		t.Fatalf("expected 1 parquet row, got %d", len(readBack))
	}
	if readBack[0].TransactionHash != transformed.TransactionHash {
		t.Fatalf("wrong parquet row read back: got transaction_hash %q, want %q", readBack[0].TransactionHash, transformed.TransactionHash)
	}

	schemaFields := map[string]bool{}
	for _, schemaElement := range parquetReader.Footer.Schema {
		schemaFields[schemaElement.GetName()] = true
	}

	if !schemaFields["ExtraSigners"] {
		t.Fatalf("parquet schema is missing ExtraSigners; schema fields=%v", schemaFields)
	}
	if schemaFields["TxSigners"] {
		t.Fatal("parquet schema unexpectedly contains TxSigners")
	}

	t.Logf("JSON output includes %d tx_signers, but the emitted parquet schema has no tx_signers column", len(transformed.TxSigners))
}
```

## Expected vs Actual Behavior

- **Expected**: The Parquet transaction export preserves the same populated `tx_signers` field that exists in the JSON transaction row.
- **Actual**: The Parquet file has no `tx_signers` column at all, so signer data is silently dropped from every exported row.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses the real transaction fixture, runs `TransformTransaction()`, writes an actual Parquet file, and inspects the emitted file schema.
2. Realistic preconditions: YES — any ordinary `export_transactions --write-parquet` run on signed transactions follows this path.
3. Bug vs by-design: BUG — the transform explicitly computes `TxSigners`, JSON exports it, and Parquet already carries other list fields such as `ExtraSigners`; there is no evidence this omission is intentional.
4. Final severity: High — this is structural export corruption affecting a non-financial field across all Parquet transaction rows.
5. In scope: YES — it is a concrete production export path producing silently incomplete data.
6. Test correctness: CORRECT — the test verifies the real file artifact rather than only reflecting on an in-memory struct.
7. Alternative explanations: NONE — even if `tx_signers` has separate semantic issues upstream, that does not explain or negate the Parquet-only field loss.
8. Novelty: NOVEL

## Suggested Fix

Add `TxSigners []string` to `TransactionOutputParquet` using the existing string-list encoding pattern and copy `to.TxSigners` into that field inside `TransactionOutput.ToParquet()`.
