# H001: Transaction parquet export drops `tx_signers`

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_transactions --write-parquet` serializes a transaction row, the parquet output should preserve the same `tx_signers` values that `TransformTransaction()` already places in JSON output. For the repository's existing transaction fixtures, rows that contain one fake `G...` signer string in `TransactionOutput.TxSigners` should produce a parquet row with that same signer list rather than silently omitting the column.

## Mechanism

`TransformTransaction()` always populates `TransactionOutput.TxSigners`, and the transaction tests assert non-empty `TxSigners` on multiple fixtures. But `TransactionOutputParquet` has no `tx_signers` field at all, and `TransactionOutput.ToParquet()` therefore has nowhere to copy the populated slice. The parquet export path silently strips signer data from every row even though the JSON export for the same command includes it.

## Trigger

Run `export_transactions` with `--write-parquet` on any normal signed transaction or fee-bump transaction. The JSON row contains `tx_signers`, but the parquet artifact has no `tx_signers` column/value for that row.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:227-300` — populates `TxSigners` for classic and fee-bump envelopes
- `internal/transform/schema.go:TransactionOutput:41-84` — JSON schema defines `TxSigners []string`
- `internal/transform/transaction_test.go:makeTransactionTestOutput:84-217` — existing expected outputs include non-empty `TxSigners`
- `internal/transform/schema_parquet.go:TransactionOutputParquet:31-74` — parquet schema omits `tx_signers`
- `internal/transform/parquet_converter.go:TransactionOutput.ToParquet:59-102` — parquet conversion never copies `TxSigners`
- `cmd/export_transactions.go:transactionsCmd.Run:51-65` — command writes parquet rows from `TransactionOutputParquet`

## Evidence

The codebase already supports repeated string fields in parquet via `ExtraSigners []string`, so the omission is not explained by a blanket "lists are unsupported" limitation. This looks like a schema/converter drift bug isolated to `tx_signers`.

## Anti-Evidence

The current `tx_signers` values are themselves known-bad identity data in JSON exports, so downstream users may already avoid the field. But the parquet path still corrupts the dataset shape further by removing the field entirely instead of matching the current JSON contract.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `TransformTransaction()` through `ToParquet()` to `WriteParquet()` in the export command. `TransactionOutput` defines `TxSigners []string` at schema.go:83, which is populated by `getTxSigners()` at transaction.go:227-270 (classic) and transaction.go:295-300 (fee-bump). However, `TransactionOutputParquet` (schema_parquet.go:31-74) has no corresponding field, and `ToParquet()` (parquet_converter.go:59-102) has no mapping for it. The export command at export_transactions.go:63-65 writes parquet via `WriteParquet()` using `TransactionOutputParquet` as the schema, so the column is silently absent.

### Code Paths Examined

- `internal/transform/schema.go:83` — confirms `TxSigners []string` field exists in JSON output struct
- `internal/transform/schema_parquet.go:31-74` — confirms NO `TxSigners` field in parquet struct; `ExtraSigners []string` IS present at line 59, proving `[]string` parquet support exists
- `internal/transform/parquet_converter.go:59-102` — confirms `ToParquet()` maps all fields EXCEPT `TxSigners`; `ExtraSigners` is mapped at line 87
- `internal/transform/transaction.go:227-270` — `getTxSigners()` called and result assigned to `TxSigners` for classic envelopes
- `internal/transform/transaction.go:295-300` — fee-bump path overwrites `TxSigners` with fee-bump signatures
- `internal/transform/transaction.go:349-360` — `getTxSigners()` encodes signature bytes as `VersionByteAccountID`
- `internal/transform/transaction_test.go:112,143,176,216` — four test fixtures assert non-empty `TxSigners`
- `cmd/export_transactions.go:51-65` — parquet write path uses `TransactionOutputParquet` schema

### Findings

The schema drift is confirmed: `TransactionOutputParquet` is missing the `TxSigners` field that `TransactionOutput` defines. The `ToParquet()` converter has no mapping for it. The parquet library uses the struct definition to determine columns, so the column is entirely absent from parquet output.

This is a genuine schema inconsistency between JSON and Parquet export formats. The pattern is identical to how `ExtraSigners` is handled (schema_parquet.go:59, parquet_converter.go:87), so adding the field follows an established pattern.

Severity downgraded from High to Medium because the confirmed success finding `001-tx-signers-encode-signatures` establishes that the values in `TxSigners` are already wrong (signature bytes encoded as fake account IDs). The parquet column is missing, but the data it would contain is itself incorrect. The practical impact is a schema contract mismatch between output formats, not new data corruption. A complete fix requires both adding the parquet column AND fixing the underlying value computation.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or create if not present; alternatively append to `internal/transform/transaction_test.go`)
- **Setup**: Construct a `TransactionOutput` with `TxSigners: []string{"GABC..."}` and call `ToParquet()`
- **Steps**: Cast the result to `TransactionOutputParquet` and inspect whether a `TxSigners` field exists via reflection, or simply verify the struct definition lacks the field
- **Assertion**: Assert that `TransactionOutputParquet` should contain a `TxSigners` field — currently it does not, confirming the schema drift. A compile-time check: `_ = TransactionOutputParquet{}.TxSigners` would fail to compile.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTransactionParquetDropsTxSigners"
**Test Language**: Go

### Demonstration

The test constructs a `TransactionOutput` with two signers in `TxSigners` and one entry in `ExtraSigners`, then calls `ToParquet()`. Using reflection, it confirms that `ExtraSigners` (also `[]string`) is correctly preserved in the parquet struct, but `TxSigners` has no corresponding field in `TransactionOutputParquet` at all. This proves the parquet schema silently drops `tx_signers` data for every transaction while the identical type `ExtraSigners` is correctly mapped — a schema drift bug.

### Test Body

```go
func TestTransactionParquetDropsTxSigners(t *testing.T) {
	signers := []string{
		"GABCDEFGHIJKLMNOPQRSTUVWXYZ234567ABCDEFGHIJKLMNOPQRST",
		"GXYZ234567ABCDEFGHIJKLMNOPQRSTUVWXYZ234567ABCDEFGHIJK",
	}

	// 1. Construct a TransactionOutput with non-empty TxSigners
	txOutput := TransactionOutput{
		TransactionHash: "abc123",
		LedgerSequence:  100,
		Account:         "GABC",
		TxSigners:       signers,
		ExtraSigners:    []string{"extra1"},
		CreatedAt:       time.Now(),
		ClosedAt:        time.Now(),
	}

	// 2. Verify JSON output includes tx_signers
	jsonBytes, err := json.Marshal(txOutput)
	if err != nil {
		t.Fatalf("Failed to marshal TransactionOutput to JSON: %v", err)
	}
	jsonStr := string(jsonBytes)
	if !strings.Contains(jsonStr, "tx_signers") {
		t.Fatalf("JSON output should contain tx_signers field, got: %s", jsonStr)
	}
	if !strings.Contains(jsonStr, signers[0]) {
		t.Fatalf("JSON output should contain signer value %s", signers[0])
	}
	t.Logf("JSON output contains tx_signers with %d signers — correct", len(signers))

	// 3. Convert to Parquet and verify tx_signers is missing
	parquetResult := txOutput.ToParquet()
	parquetVal := reflect.ValueOf(parquetResult)
	parquetType := parquetVal.Type()

	// Check that ExtraSigners IS present (proving []string fields can exist in parquet)
	extraSignersField := parquetVal.FieldByName("ExtraSigners")
	if !extraSignersField.IsValid() {
		t.Fatalf("ExtraSigners field should exist in TransactionOutputParquet")
	}
	extraSlice, ok := extraSignersField.Interface().([]string)
	if !ok || len(extraSlice) != 1 || extraSlice[0] != "extra1" {
		t.Fatalf("ExtraSigners should be preserved in parquet, got %v", extraSignersField.Interface())
	}
	t.Logf("ExtraSigners correctly preserved in parquet: %v", extraSlice)

	// Check that TxSigners is NOT present — this is the bug
	txSignersField := parquetVal.FieldByName("TxSigners")
	if txSignersField.IsValid() {
		t.Logf("TxSigners field found in parquet output — bug may be fixed")
	} else {
		t.Errorf("SCHEMA DRIFT CONFIRMED: TransactionOutputParquet has no TxSigners field")
		t.Errorf("TransactionOutput.TxSigners = %v (len=%d)", signers, len(signers))
		t.Errorf("TransactionOutputParquet has %d fields but none named TxSigners", parquetType.NumField())
		t.Errorf("ExtraSigners ([]string) IS mapped — so []string support exists in the parquet schema")
		t.Errorf("Result: parquet export silently drops tx_signers data for every transaction")
	}

	// 4. Enumerate all parquet fields to show TxSigners is missing
	fieldNames := make([]string, parquetType.NumField())
	for i := 0; i < parquetType.NumField(); i++ {
		fieldNames[i] = parquetType.Field(i).Name
	}
	hasTxSigners := false
	for _, name := range fieldNames {
		if name == "TxSigners" {
			hasTxSigners = true
			break
		}
	}
	if !hasTxSigners {
		t.Logf("BUG CONFIRMED: TxSigners not in parquet field list: %v", fieldNames)
	}
}
```

### Test Output

```
=== RUN   TestTransactionParquetDropsTxSigners
    data_integrity_poc_test.go:289: JSON output contains tx_signers with 2 signers — correct
    data_integrity_poc_test.go:305: ExtraSigners correctly preserved in parquet: [extra1]
    data_integrity_poc_test.go:312: SCHEMA DRIFT CONFIRMED: TransactionOutputParquet has no TxSigners field
    data_integrity_poc_test.go:313: TransactionOutput.TxSigners = [GABCDEFGHIJKLMNOPQRSTUVWXYZ234567ABCDEFGHIJKLMNOPQRST GXYZ234567ABCDEFGHIJKLMNOPQRSTUVWXYZ234567ABCDEFGHIJK] (len=2)
    data_integrity_poc_test.go:314: TransactionOutputParquet has 41 fields but none named TxSigners
    data_integrity_poc_test.go:315: ExtraSigners ([]string) IS mapped — so []string support exists in the parquet schema
    data_integrity_poc_test.go:316: Result: parquet export silently drops tx_signers data for every transaction
    data_integrity_poc_test.go:332: BUG CONFIRMED: TxSigners not in parquet field list: [TransactionHash LedgerSequence Account AccountMuxed AccountSequence MaxFee FeeCharged OperationCount TxEnvelope TxResult TxMeta TxFeeMeta CreatedAt MemoType Memo TimeBounds Successful TransactionID FeeAccount FeeAccountMuxed InnerTransactionHash NewMaxFee LedgerBounds MinAccountSequence MinAccountSequenceAge MinAccountSequenceLedgerGap ExtraSigners ClosedAt ResourceFee SorobanResourcesInstructions SorobanResourcesReadBytes SorobanResourcesDiskReadBytes SorobanResourcesWriteBytes SorobanResourcesArchivedEntries TransactionResultCode InclusionFeeBid InclusionFeeCharged ResourceFeeRefund TotalNonRefundableResourceFeeCharged TotalRefundableResourceFeeCharged RentFeeCharged]
--- FAIL: TestTransactionParquetDropsTxSigners (0.00s)
FAIL
```
