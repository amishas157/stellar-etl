# H002: Transaction parquet fabricates absent optional fields

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For transactions that do not carry optional muxed-account or fee-bump metadata, the Parquet export should preserve those columns as null / absent, matching the JSON output. In particular, classic non-fee-bump transactions should not acquire synthetic values for `account_muxed`, `fee_account`, `fee_account_muxed`, `inner_transaction_hash`, or `new_max_fee`.

## Mechanism

`TransformTransaction()` only populates these fields in the muxed-account and fee-bump branches, and the JSON schema marks them optional with `omitempty`. The Parquet schema instead defines them as required `string` / `int64` columns, and `ToParquet()` copies zero values directly, so classic rows are exported with fabricated `""` and `0` values that look like real data rather than missing data.

## Trigger

Export any classic transaction whose source account is not muxed and whose envelope is not fee-bump. The JSON row omits these fields, while the Parquet row serializes empty strings and `new_max_fee = 0`.

## Target Code

- `internal/transform/transaction.go:273-301` — only sets muxed-account and fee-bump fields inside conditional branches
- `internal/transform/schema.go:45,60-63` — JSON schema declares these fields optional with `omitempty`
- `internal/transform/schema_parquet.go:36,51-55` — Parquet schema forces concrete string / int64 columns
- `internal/transform/parquet_converter.go:64,79-83` — converter writes zero values into Parquet rows

## Evidence

The transaction test fixture's first classic transaction leaves all fee-bump-only fields unset, which matches the JSON contract. The Parquet path has no nullable wrapper for those same fields, so absent metadata is converted into concrete empty values during serialization.

## Anti-Evidence

An empty string is not a valid Stellar account or transaction hash, so downstream consumers can sometimes infer that the value is synthetic. But that still changes the output contract from "missing" to "present with a value," which breaks null-sensitive analytics and is the same class of corruption already proven elsewhere in the repository for other optional columns.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Success finding 003 covers `min_account_sequence` (null.Int fields); 010 covers `sponsor` (null.String fields); 011 covers effects `address_muxed` (null.String). This hypothesis covers a distinct set of transaction fields using plain string/int64 with `omitempty`.

### Trace Summary

`TransformTransaction()` in `transaction.go` constructs a `TransactionOutput` struct with all fields initialized to Go zero values. The fee-bump/muxed fields (`AccountMuxed`, `FeeAccount`, `FeeAccountMuxed`, `InnerTransactionHash`, `NewMaxFee`) are only populated inside conditional branches (lines 274-301) that check for muxed source accounts and fee-bump envelopes. For classic transactions, these remain at zero values ("" and 0). The JSON schema uses `omitempty` tags to correctly omit these, but `ToParquet()` copies the zero values directly into required Parquet columns, fabricating present-but-empty data.

### Code Paths Examined

- `internal/transform/transaction.go:273-281` — `AccountMuxed` is only set when `sourceAccount.Type == xdr.CryptoKeyTypeKeyTypeMuxedEd25519`; otherwise stays ""
- `internal/transform/transaction.go:284-301` — `FeeAccount`, `FeeAccountMuxed`, `InnerTransactionHash`, `NewMaxFee` are only set inside `transaction.Envelope.IsFeeBump()` block; otherwise stay at zero values
- `internal/transform/schema.go:45,60-63` — JSON tags `account_muxed,omitempty`, `fee_account,omitempty`, `fee_account_muxed,omitempty`, `inner_transaction_hash,omitempty`, `new_max_fee,omitempty` correctly omit zero values
- `internal/transform/schema_parquet.go:36,51-54` — Parquet schema defines all five as required (non-optional) columns: `string` for text fields, `int64` for `NewMaxFee`
- `internal/transform/parquet_converter.go:64,79-82` — `ToParquet()` directly copies: `to.AccountMuxed`, `to.FeeAccount`, `to.FeeAccountMuxed`, `to.InnerTransactionHash`, `int64(to.NewMaxFee)` — no null/optional handling

### Findings

The bug is confirmed and follows the exact same pattern as three already-confirmed findings (003, 010, 011) in this codebase. For classic (non-fee-bump, non-muxed) transactions:

1. **JSON output**: Fields are absent (omitted by `omitempty`) — correct behavior
2. **Parquet output**: Fields contain `""` (strings) or `0` (NewMaxFee) — fabricated data

The five affected columns are:
- `account_muxed`: "" instead of null (affects ~99%+ of transactions since muxed accounts are rare)
- `fee_account`: "" instead of null (affects all non-fee-bump transactions)
- `fee_account_muxed`: "" instead of null (affects all non-fee-bump transactions and fee-bumps with non-muxed fee accounts)
- `inner_transaction_hash`: "" instead of null (affects all non-fee-bump transactions)
- `new_max_fee`: 0 instead of null (affects all non-fee-bump transactions)

This breaks null-sensitive analytics. For example, `SELECT COUNT(*) FROM transactions WHERE fee_account IS NOT NULL` would count ALL transactions instead of only fee-bump transactions, because `""` is not NULL.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go` (append to existing file if it exists, otherwise create)
- **Setup**: Use `makeTransactionTestInput()` to get a classic (non-fee-bump) transaction fixture. Transform it with `TransformTransaction()`.
- **Steps**: (1) Confirm the JSON-layer output has zero-value fields that would be omitted by `omitempty`. (2) Call `ToParquet()` on the output. (3) Cast the result to `TransactionOutputParquet`.
- **Assertion**: Assert that `parquet.FeeAccount == ""` and `parquet.NewMaxFee == 0` — demonstrating that absent optional fields are fabricated as concrete values in Parquet output. Contrast this with the JSON behavior where these fields would be omitted entirely.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTransactionParquetFabricatesOptionalFields"
**Test Language**: Go

### Demonstration

The test transforms a classic (non-fee-bump, non-muxed) transaction through the production `TransformTransaction()` path, then proves the JSON/Parquet divergence: JSON marshaling correctly omits all 5 optional fields (`account_muxed`, `fee_account`, `fee_account_muxed`, `inner_transaction_hash`, `new_max_fee`) via `omitempty`, but `ToParquet()` fabricates them as concrete `""` and `0` values. This confirms that Parquet consumers receive present-but-empty data where they should see NULL, breaking null-sensitive analytics.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"
)

// TestTransactionParquetFabricatesOptionalFields demonstrates that for classic
// (non-fee-bump, non-muxed) transactions, the Parquet conversion fabricates
// concrete zero values for fields that the JSON schema correctly omits via omitempty.
func TestTransactionParquetFabricatesOptionalFields(t *testing.T) {
	// Step 1: Get a classic (non-fee-bump) transaction from the existing test fixtures
	hardCodedTransactions, hardCodedHeaders, err := makeTransactionTestInput()
	if err != nil {
		t.Fatalf("makeTransactionTestInput() failed: %v", err)
	}

	// The first transaction in the fixture is a classic (non-fee-bump) transaction
	classicTx := hardCodedTransactions[0]
	classicHeader := hardCodedHeaders[0]

	// Verify precondition: this is NOT a fee-bump transaction
	if classicTx.Envelope.IsFeeBump() {
		t.Fatal("Expected first test transaction to be classic (non-fee-bump)")
	}

	// Step 2: Transform the transaction through the production code path
	output, err := TransformTransaction(classicTx, classicHeader)
	if err != nil {
		t.Fatalf("TransformTransaction() failed: %v", err)
	}

	// Step 3: Verify that the JSON representation correctly omits these fields.
	// With omitempty, zero-value strings and int64 should not appear in JSON.
	jsonBytes, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("json.Marshal() failed: %v", err)
	}
	var jsonMap map[string]interface{}
	if err := json.Unmarshal(jsonBytes, &jsonMap); err != nil {
		t.Fatalf("json.Unmarshal() failed: %v", err)
	}

	// These five fields should be ABSENT from JSON due to omitempty
	omittedFields := []string{
		"account_muxed",
		"fee_account",
		"fee_account_muxed",
		"inner_transaction_hash",
		"new_max_fee",
	}
	for _, field := range omittedFields {
		if _, exists := jsonMap[field]; exists {
			t.Errorf("JSON output should omit %q for classic transaction, but it is present", field)
		}
	}

	// Step 4: Convert to Parquet and verify that these fields are fabricated
	// as concrete zero values instead of being absent/null.
	parquetIface := output.ToParquet()
	parquet, ok := parquetIface.(TransactionOutputParquet)
	if !ok {
		t.Fatalf("ToParquet() did not return TransactionOutputParquet, got %T", parquetIface)
	}

	// These fields should be null/absent for a classic transaction, but Parquet
	// fabricates concrete zero values — this is the bug.
	if parquet.AccountMuxed != "" {
		t.Errorf("Expected AccountMuxed to be empty string (fabricated), got %q", parquet.AccountMuxed)
	}
	if parquet.FeeAccount != "" {
		t.Errorf("Expected FeeAccount to be empty string (fabricated), got %q", parquet.FeeAccount)
	}
	if parquet.FeeAccountMuxed != "" {
		t.Errorf("Expected FeeAccountMuxed to be empty string (fabricated), got %q", parquet.FeeAccountMuxed)
	}
	if parquet.InnerTransactionHash != "" {
		t.Errorf("Expected InnerTransactionHash to be empty string (fabricated), got %q", parquet.InnerTransactionHash)
	}
	if parquet.NewMaxFee != 0 {
		t.Errorf("Expected NewMaxFee to be 0 (fabricated), got %d", parquet.NewMaxFee)
	}

	// SUMMARY: The JSON output correctly omits these 5 optional fields (via omitempty),
	// but the Parquet output contains fabricated concrete values ("" and 0).
	// This means Parquet consumers see present-but-empty data where they should see NULL,
	// breaking null-sensitive analytics (e.g., COUNT WHERE fee_account IS NOT NULL).
	t.Logf("BUG CONFIRMED: JSON omits %d optional fields for classic transactions,", len(omittedFields))
	t.Logf("but Parquet fabricates them as concrete zero values:")
	t.Logf("  AccountMuxed = %q (should be null)", parquet.AccountMuxed)
	t.Logf("  FeeAccount = %q (should be null)", parquet.FeeAccount)
	t.Logf("  FeeAccountMuxed = %q (should be null)", parquet.FeeAccountMuxed)
	t.Logf("  InnerTransactionHash = %q (should be null)", parquet.InnerTransactionHash)
	t.Logf("  NewMaxFee = %d (should be null)", parquet.NewMaxFee)
}
```

### Test Output

```
=== RUN   TestTransactionParquetFabricatesOptionalFields
    data_integrity_poc_test.go:88: BUG CONFIRMED: JSON omits 5 optional fields for classic transactions,
    data_integrity_poc_test.go:89: but Parquet fabricates them as concrete zero values:
    data_integrity_poc_test.go:90:   AccountMuxed = "" (should be null)
    data_integrity_poc_test.go:91:   FeeAccount = "" (should be null)
    data_integrity_poc_test.go:92:   FeeAccountMuxed = "" (should be null)
    data_integrity_poc_test.go:93:   InnerTransactionHash = "" (should be null)
    data_integrity_poc_test.go:94:   NewMaxFee = 0 (should be null)
--- PASS: TestTransactionParquetFabricatesOptionalFields (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.916s
```
