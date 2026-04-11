# 012: Transaction Parquet export drops `tx_signers`

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` populates `TransactionOutput.TxSigners` for signed transactions, and the JSON export path preserves that slice. The Parquet export path then silently discards it because `TransactionOutputParquet` has no `tx_signers` field and `TransactionOutput.ToParquet()` never maps one.

As a result, `export_transactions --write-parquet` emits rows that are structurally inconsistent with the JSON export for the same transaction set: signer data present in JSON is absent from every Parquet row.

## Root Cause

The transaction JSON schema and transform logic carry `TxSigners`, but the Parquet schema/converter drifted. `TransactionOutputParquet` stops at `rent_fee_charged`, and `TransactionOutput.ToParquet()` copies every nearby field except `TxSigners`, so `WriteParquet()` has no column available for the populated signer slice.

## Reproduction

During normal `export_transactions --write-parquet` execution, `TransformTransaction()` extracts `tx_signers` from the transaction envelope and returns them in `TransactionOutput`. JSON export writes that struct directly, but the Parquet path converts through `TransactionOutputParquet`, where the signer slice is omitted entirely.

## Affected Code

- `internal/transform/transaction.go:TransformTransaction:227-300` — populates `TxSigners` for classic and fee-bump transactions
- `internal/transform/schema.go:TransactionOutput:41-84` — JSON transaction schema defines `TxSigners []string`
- `internal/transform/schema_parquet.go:TransactionOutputParquet:31-74` — Parquet transaction schema omits `tx_signers`
- `internal/transform/parquet_converter.go:TransactionOutput.ToParquet:59-102` — converter maps adjacent fields but never serializes `TxSigners`
- `cmd/export_transactions.go:transactionsCmd.Run:51-65` — Parquet export writes rows using `TransactionOutputParquet`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTransactionParquetDropsTxSigners`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Append the test body below to `internal/transform/data_integrity_poc_test.go` and add `reflect` to the imports if needed
  3. Run `go test ./internal/transform/... -run TestTransactionParquetDropsTxSigners -v`
  4. Observe that `TransformTransaction()` produces non-empty `tx_signers`, JSON includes them, and the test fails because `TransactionOutputParquet` has no `TxSigners` field

### Test Body

```go
func TestTransactionParquetDropsTxSigners(t *testing.T) {
	inputs, headers, err := makeTransactionTestInput()
	if err != nil {
		t.Fatalf("makeTransactionTestInput() returned error: %v", err)
	}

	expected, err := makeTransactionTestOutput()
	if err != nil {
		t.Fatalf("makeTransactionTestOutput() returned error: %v", err)
	}

	if len(inputs) == 0 || len(headers) == 0 || len(expected) == 0 {
		t.Fatalf("transaction test fixtures are unexpectedly empty")
	}

	transformed, err := TransformTransaction(inputs[0], headers[0])
	if err != nil {
		t.Fatalf("TransformTransaction() returned error: %v", err)
	}

	if len(transformed.TxSigners) == 0 {
		t.Fatalf("fixture precondition failed: TransformTransaction() returned no tx_signers")
	}

	if !reflect.DeepEqual(transformed.TxSigners, expected[0].TxSigners) {
		t.Fatalf("fixture drift: TransformTransaction() tx_signers = %v, expected %v", transformed.TxSigners, expected[0].TxSigners)
	}

	jsonBytes, err := json.Marshal(transformed)
	if err != nil {
		t.Fatalf("json.Marshal(transformed) returned error: %v", err)
	}

	jsonOutput := string(jsonBytes)
	if !strings.Contains(jsonOutput, `"tx_signers"`) {
		t.Fatalf("JSON output should include tx_signers, got %s", jsonOutput)
	}
	if !strings.Contains(jsonOutput, transformed.TxSigners[0]) {
		t.Fatalf("JSON output should include signer %q, got %s", transformed.TxSigners[0], jsonOutput)
	}

	parquetOutput, ok := transformed.ToParquet().(TransactionOutputParquet)
	if !ok {
		t.Fatalf("ToParquet() returned unexpected type %T", transformed.ToParquet())
	}

	if _, found := reflect.TypeOf(parquetOutput).FieldByName("ExtraSigners"); !found {
		t.Fatalf("TransactionOutputParquet should expose ExtraSigners; repeated string fields are supported")
	}

	if _, found := reflect.TypeOf(parquetOutput).FieldByName("TxSigners"); found {
		t.Fatalf("expected TransactionOutputParquet to omit TxSigners for this PoC, but field exists")
	}

	t.Fatalf("PARQUET DROP CONFIRMED: TransformTransaction produced tx_signers=%v, but TransactionOutputParquet has no TxSigners field so the parquet export path cannot serialize them", transformed.TxSigners)
}
```

## Expected vs Actual Behavior

- **Expected**: When a transaction row has populated `tx_signers` in `TransactionOutput`, the Parquet export should preserve the same signer list in a `tx_signers` column.
- **Actual**: `TransactionOutputParquet` has no `tx_signers` field, so the Parquet path drops the signer list from every row even though the JSON export for the same row includes it.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the real `TransformTransaction()` fixture path, confirms `tx_signers` is populated, then converts through the real `ToParquet()` implementation.
2. Realistic preconditions: YES — any ordinary signed transaction or fee-bump transaction exported with `--write-parquet` reaches this code path.
3. Bug vs by-design: BUG — adjacent repeated string data (`ExtraSigners`) is supported in the Parquet schema, and nothing in the code or docs indicates `tx_signers` should be dropped while JSON keeps it.
4. Final severity: High — this is structural data corruption in every transaction Parquet export row, not just a presentation difference.
5. In scope: YES — the omission is in stellar-etl's own schema/converter/export code, not upstream SDK behavior.
6. Test correctness: CORRECT — the test uses production fixtures, production transform code, and a schema inspection that directly explains why Parquet cannot serialize the populated field.
7. Alternative explanations: NONE
8. Novelty: NOT ASSESSED — this review confirms the bug is real; duplicate handling is orchestrator-owned.

## Suggested Fix

Add `TxSigners []string` to `TransactionOutputParquet` using the same Parquet list tag pattern as `ExtraSigners`, and map `to.TxSigners` inside `TransactionOutput.ToParquet()`. The separate `tx_signers` value-generation bug should still be fixed independently so both JSON and Parquet exports carry correct signer identities.
