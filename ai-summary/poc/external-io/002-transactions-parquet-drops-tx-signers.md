# H002: Transaction Parquet drops `tx_signers`

**Date**: 2026-04-10
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_transactions` should emit the same signer set in both JSON and Parquet outputs. For classic transactions and fee-bump transactions alike, downstream Parquet readers should be able to see the `tx_signers` array that `TransformTransaction()` derives from the decorated signatures.

## Mechanism

`TransformTransaction()` fills `TxSigners`, and the JSON schema exposes that field as `tx_signers`. The Parquet path never carries it across: `TransactionOutputParquet` has no `tx_signers` column and `TransactionOutput.ToParquet()` has no corresponding assignment, so Parquet exports silently drop signer data that exists in JSON.

## Trigger

Run `export_transactions --write-parquet` on any ledger range containing a transaction with one or more signatures, especially a fee-bump transaction whose signer set is operationally important. The JSON export will include `tx_signers`, while the Parquet file will have no equivalent column.

## Target Code

- `internal/transform/transaction.go:227-300` — transaction transform populates `TxSigners`
- `internal/transform/schema.go:41-84` — JSON schema includes `tx_signers`
- `internal/transform/schema_parquet.go:31-74` — Parquet schema has no `tx_signers` field
- `internal/transform/parquet_converter.go:59-103` — `ToParquet()` never copies signer data
- `cmd/export_transactions.go:63-65` — this command writes the lossy Parquet output

## Evidence

`getTxSigners()` is called in both classic and fee-bump branches, and the transformed struct assigns `TxSigners` before returning. The JSON schema retains the field, but the Parquet schema/converter omit it entirely, so the exported Parquet shape cannot represent the source value.

## Anti-Evidence

The JSON export is correct, so users consuming line-delimited JSON are unaffected. This is specifically a Parquet consistency bug.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformTransaction()` in `transaction.go` populates `TxSigners` at line 270 (classic path) and line 300 (fee-bump path) by calling `getTxSigners()` on the envelope signatures. The JSON schema (`TransactionOutput` in `schema.go:83`) includes `TxSigners []string` with json tag `tx_signers`, so JSON output is complete. However, `TransactionOutputParquet` in `schema_parquet.go:32-74` has no `TxSigners` field at all, and the `ToParquet()` converter at `parquet_converter.go:59-103` maps every other field but silently drops `TxSigners`. Any downstream consumer reading Parquet output will see no signer data.

### Code Paths Examined

- `internal/transform/transaction.go:227-300` — Confirmed `TxSigners` is populated in both classic (line 270) and fee-bump (line 295-300) branches via `getTxSigners()`
- `internal/transform/schema.go:83` — Confirmed `TxSigners []string \`json:"tx_signers"\`` is present in `TransactionOutput`
- `internal/transform/schema_parquet.go:32-74` — Confirmed `TransactionOutputParquet` has NO `TxSigners` field; all other fields from `TransactionOutput` are present
- `internal/transform/parquet_converter.go:59-103` — Confirmed `ToParquet()` maps 33 fields but omits `TxSigners`; the function returns `TransactionOutputParquet` which structurally cannot hold signer data
- `cmd/export_transactions.go:63-65` — Confirmed Parquet write path calls `WriteParquet(transformedTransaction, parquetPath, new(transform.TransactionOutputParquet))`

### Findings

The bug is a straightforward omission: when `TransactionOutputParquet` was defined, the `TxSigners` field was not included in the Parquet schema struct. Consequently, the `ToParquet()` converter has no target field to assign it to. This creates a silent data discrepancy where JSON exports contain full signer information but Parquet exports contain none.

The omission affects every transaction in every Parquet export — not just an edge case. Since every Stellar transaction has at least one signature, every row in the Parquet output is missing signer data. This is particularly impactful for fee-bump transactions where the signer set is updated at line 300 to reflect the fee-bump envelope signatures.

Note: `ExtraSigners` (a separate precondition field) IS correctly carried through to Parquet at `schema_parquet.go:59` and `parquet_converter.go:87`. Only the actual transaction signers (`TxSigners`) are dropped.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go` (append a new test)
- **Setup**: Use an existing transaction test fixture that produces a `TransactionOutput` with a non-empty `TxSigners` slice
- **Steps**: Call `TransformTransaction()` to get a `TransactionOutput`, then call `.ToParquet()` on the result and cast to `TransactionOutputParquet`
- **Assertion**: Verify that the `TransactionOutputParquet` struct has no `TxSigners` field (compile-time check) — or more practically, compare the JSON-marshalled output of both structs and show that `tx_signers` is present in JSON but absent from Parquet. A field-count comparison between `TransactionOutput` and `TransactionOutputParquet` would also demonstrate the mismatch.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestParquetDropsTxSigners"
**Test Language**: Go

### Demonstration

The test constructs a `TransactionOutput` with two signers in `TxSigners`, verifies they appear in JSON-marshalled output, then calls `ToParquet()` and confirms the resulting `TransactionOutputParquet` struct has no `TxSigners` field via reflection and no signer data in its JSON representation. This proves every Parquet export silently drops all transaction signer data.

### Test Body

```go
package transform

import (
	"encoding/json"
	"reflect"
	"testing"
	"time"

	"github.com/guregu/null"
)

// TestParquetDropsTxSigners demonstrates that TransactionOutputParquet
// silently drops the TxSigners field present in TransactionOutput.
// The JSON output includes tx_signers; the Parquet struct omits it entirely.
func TestParquetDropsTxSigners(t *testing.T) {
	// 1. Construct a TransactionOutput with non-empty TxSigners
	txOutput := TransactionOutput{
		TransactionHash: "abc123",
		LedgerSequence:  100,
		Account:         "GABC",
		AccountSequence: 1,
		MaxFee:          100,
		FeeCharged:      50,
		OperationCount:  1,
		TxEnvelope:      "env",
		TxResult:        "res",
		TxMeta:          "meta",
		TxFeeMeta:       "feemeta",
		CreatedAt:       time.Unix(1000, 0),
		MemoType:        "none",
		TimeBounds:      "",
		Successful:      true,
		TransactionID:   42,
		MinAccountSequence:          null.IntFrom(0),
		MinAccountSequenceAge:       null.IntFrom(0),
		MinAccountSequenceLedgerGap: null.IntFrom(0),
		ClosedAt:                    time.Unix(1000, 0),
		TxSigners:                   []string{"GABC...SIGNER1", "GDEF...SIGNER2"},
	}

	// 2. Verify TxSigners is present in JSON output
	jsonBytes, err := json.Marshal(txOutput)
	if err != nil {
		t.Fatalf("Failed to marshal TransactionOutput to JSON: %v", err)
	}
	var jsonMap map[string]interface{}
	if err := json.Unmarshal(jsonBytes, &jsonMap); err != nil {
		t.Fatalf("Failed to unmarshal JSON: %v", err)
	}
	signers, hasTxSigners := jsonMap["tx_signers"]
	if !hasTxSigners {
		t.Fatal("tx_signers missing from JSON output — test setup is wrong")
	}
	signerList, ok := signers.([]interface{})
	if !ok || len(signerList) != 2 {
		t.Fatalf("Expected 2 tx_signers in JSON, got %v", signers)
	}

	// 3. Convert to Parquet and check for TxSigners field
	parquetResult := txOutput.ToParquet()
	parquetType := reflect.TypeOf(parquetResult)

	// The Parquet struct should have a TxSigners field if signers are preserved.
	// This demonstrates the field is structurally absent.
	_, hasTxSignersField := parquetType.FieldByName("TxSigners")

	// 4. Also marshal the Parquet struct to JSON to show the data loss
	parquetBytes, err := json.Marshal(parquetResult)
	if err != nil {
		t.Fatalf("Failed to marshal Parquet struct to JSON: %v", err)
	}
	var parquetMap map[string]interface{}
	if err := json.Unmarshal(parquetBytes, &parquetMap); err != nil {
		t.Fatalf("Failed to unmarshal Parquet JSON: %v", err)
	}

	// Check that no key in the Parquet JSON output contains signer data
	_, parquetHasSigners := parquetMap["tx_signers"]
	_, parquetHasSigners2 := parquetMap["TxSigners"]

	// --- Assertions ---
	// The Parquet struct has no TxSigners field at all
	if hasTxSignersField {
		t.Error("UNEXPECTED: TransactionOutputParquet has a TxSigners field — hypothesis would be disproven")
	} else {
		t.Log("CONFIRMED: TransactionOutputParquet struct has NO TxSigners field")
	}

	// The Parquet JSON output has no signer data
	if parquetHasSigners || parquetHasSigners2 {
		t.Error("UNEXPECTED: Parquet JSON output contains signer data")
	} else {
		t.Log("CONFIRMED: Parquet JSON output contains no tx_signers data")
	}

	// The JSON output has signer data but Parquet does not — data loss proven
	if hasTxSigners && !parquetHasSigners && !hasTxSignersField {
		t.Log("BUG DEMONSTRATED: TransactionOutput.TxSigners is populated with 2 signers, " +
			"JSON output includes tx_signers, but ToParquet() produces a struct with no TxSigners field. " +
			"Parquet exports silently drop all transaction signer data.")
	} else {
		t.Error("Could not demonstrate the data loss bug")
	}
}
```

### Test Output

```
=== RUN   TestParquetDropsTxSigners
    data_integrity_poc_test.go:86: CONFIRMED: TransactionOutputParquet struct has NO TxSigners field
    data_integrity_poc_test.go:93: CONFIRMED: Parquet JSON output contains no tx_signers data
    data_integrity_poc_test.go:98: BUG DEMONSTRATED: TransactionOutput.TxSigners is populated with 2 signers, JSON output includes tx_signers, but ToParquet() produces a struct with no TxSigners field. Parquet exports silently drop all transaction signer data.
--- PASS: TestParquetDropsTxSigners (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.683s
```
