# H002: Transaction Parquet export collapses nullable preconditions to zero

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`min_account_sequence`, `min_account_sequence_age`, and `min_account_sequence_ledger_gap` should preserve the JSON transform's distinction between "precondition absent" and "precondition explicitly set to 0". Downstream analytics should be able to tell whether a transaction had no sequence preconditions at all or intentionally used zero-valued ones.

## Mechanism

The JSON schema models all three fields as `null.Int`, and `TransformTransaction()` leaves them invalid unless the envelope actually carries those preconditions. The Parquet schema narrows them to plain `int64`, and `TransactionOutput.ToParquet()` writes `.Int64` directly, so both `null.Int{}` and `null.IntFrom(0)` serialize as `0`. That silently changes the meaning of absent preconditions into present zero-valued preconditions.

## Trigger

Compare two exported transactions: one with no min-sequence preconditions and one with `minSeqAge=0` or `minSeqLedgerGap=0`. Their Parquet rows will both show `0`, even though the JSON transform distinguishes them.

## Target Code

- `internal/transform/transaction.go:113-129` — only populates the fields when the envelope exposes those preconditions
- `internal/transform/schema.go:64-67` — JSON output keeps the three fields nullable
- `internal/transform/schema_parquet.go:54-58` — Parquet schema stores them as non-null `int64`
- `internal/transform/parquet_converter.go:81-86` — converter serializes `.Int64` for all three nullable values
- `internal/transform/transaction_test.go:167-168` — tests already cover a transaction with explicit zero-valued preconditions

## Evidence

`TransformTransaction()` initializes `outputMinSequence`, `outputMinSequenceAge`, and `outputMinSequenceLedgerGap` as empty `null.Int{}` values and only assigns them when the corresponding XDR pointers are non-nil. Yet `TransactionOutput.ToParquet()` writes `to.MinAccountSequence.Int64`, `to.MinAccountSequenceAge.Int64`, and `to.MinAccountSequenceLedgerGap.Int64` into non-null Parquet columns. The test fixture at `transaction_test.go:167-168` proves that `0` is a legitimate explicit value, so collapsing null to `0` loses real semantics.

## Anti-Evidence

If downstream Parquet consumers intentionally treat `0` as a stand-in for null, this may be tolerated operationally. But the JSON schema and transform logic already encode a distinct null state, so the current Parquet conversion is still exporting a wrong value relative to the transform's own contract.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full lifecycle of the three precondition fields (`min_account_sequence`, `min_account_sequence_age`, `min_account_sequence_ledger_gap`) from XDR extraction through JSON schema to Parquet serialization. Confirmed that `TransformTransaction()` (transaction.go:113-129) uses `null.Int{}` for absent preconditions and `null.IntFrom(value)` for present ones, preserving null semantics. The Parquet converter (parquet_converter.go:84-86) reads `.Int64` directly from `null.Int`, which yields `0` for both absent (`Valid=false, Int64=0`) and explicitly-zero (`Valid=true, Int64=0`) values. The Parquet schema (schema_parquet.go:56-58) uses plain `int64` with no `OPTIONAL` repetition type. The existing test at transaction_test.go:167-168 proves `0` is a legitimate explicit value used in real transactions.

### Code Paths Examined

- `internal/transform/transaction.go:113-129` — Confirmed: `null.Int{}` (absent) vs `null.IntFrom(int64(*ptr))` (present). Three independent nil checks for MinSeqNum, MinSeqAge, MinSeqLedgerGap.
- `internal/transform/schema.go:65-67` — Confirmed: All three fields are `null.Int` in the JSON output struct.
- `internal/transform/schema_parquet.go:56-58` — Confirmed: All three fields are plain `int64` with no `repetitiontype=OPTIONAL` tag. No `*int64` pointer type used.
- `internal/transform/parquet_converter.go:84-86` — Confirmed: Reads `to.MinAccountSequence.Int64`, `to.MinAccountSequenceAge.Int64`, `to.MinAccountSequenceLedgerGap.Int64` directly. No check of `.Valid` field.
- `internal/transform/transaction_test.go:167-168` — Confirmed: Test case 3 uses `null.IntFrom(0)` for `MinAccountSequenceAge` and `MinAccountSequenceLedgerGap`, proving `0` is a legitimate explicit precondition value.
- `internal/transform/parquet_converter.go:112-113,266-272` — Confirmed: Same `.Int64` extraction pattern is used for other `null.Int` fields (`SequenceLedger`, `SequenceTime`, `SellingOfferID`, `BuyingOfferID`, `LiquidityPoolFee`, `RoundingSlippage`), suggesting a systemic issue.
- `go.mod:19` — Library is `xitongsys/parquet-go v1.6.2`, which supports nullable fields via `*int64` pointer types with `repetitiontype=OPTIONAL`.

### Findings

The bug is confirmed. When a transaction has no sequence preconditions (classic V0/V1 transactions, or V2 transactions that omit these optional fields), `TransformTransaction()` correctly produces `null.Int{}` with `Valid=false`. When a transaction explicitly sets a precondition to `0`, it produces `null.IntFrom(0)` with `Valid=true`. Both have `Int64=0`.

The Parquet converter unconditionally reads `.Int64`, collapsing both states to `0`. This means:
- A V0 transaction with no preconditions: Parquet shows `min_account_sequence_age=0`
- A V2 transaction with `minSeqAge=0`: Parquet shows `min_account_sequence_age=0`
- These are semantically different but indistinguishable in Parquet output

The JSON output correctly distinguishes them (null vs 0), so the bug is specifically in the Parquet conversion layer.

This is a systemic pattern — the same null-collapsing issue affects at least 7 other `null.Int` fields in `parquet_converter.go` (lines 112-113, 266-272). The hypothesis correctly focuses on the transaction precondition fields as the highest-impact instance, since precondition presence/absence is a meaningful analytical signal.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or `internal/transform/transaction_test.go` if no converter-specific test file exists)
- **Setup**: Create two `TransactionOutput` structs: one with `MinAccountSequenceAge: null.Int{}` (absent) and one with `MinAccountSequenceAge: null.IntFrom(0)` (explicitly zero).
- **Steps**: Call `.ToParquet()` on both structs and compare the `MinAccountSequenceAge` field of the resulting Parquet structs.
- **Assertion**: Assert that the two Parquet outputs are distinguishable. Currently they are both `0`, demonstrating the bug. The fix would use `*int64` pointer types in the Parquet schema with `repetitiontype=OPTIONAL`, setting `nil` for absent and `&zero` for explicit zero.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestParquetCollapsesNullPreconditions"
**Test Language**: Go

### Demonstration

The test constructs two `TransactionOutput` structs that differ only in whether precondition fields are absent (`null.Int{}`, Valid=false) or explicitly zero (`null.IntFrom(0)`, Valid=true). After calling `.ToParquet()` on both, all three precondition fields (`MinAccountSequence`, `MinAccountSequenceAge`, `MinAccountSequenceLedgerGap`) produce identical `int64(0)` values, proving the Parquet conversion destroys the null/zero distinction that the JSON schema correctly preserves.

### Test Body

```go
package transform

import (
	"testing"
	"time"

	"github.com/guregu/null"
)

// TestParquetCollapsesNullPreconditions demonstrates that the Parquet
// conversion of TransactionOutput collapses absent (null) precondition fields
// and explicitly-zero precondition fields into the same value (0), losing the
// distinction preserved by the JSON schema's null.Int type.
func TestParquetCollapsesNullPreconditions(t *testing.T) {
	// 1. Construct two TransactionOutput structs that differ only in
	//    precondition null-ness.
	absentPreconditions := TransactionOutput{
		MinAccountSequence:          null.Int{},          // absent — Valid=false, Int64=0
		MinAccountSequenceAge:       null.Int{},          // absent
		MinAccountSequenceLedgerGap: null.Int{},          // absent
		ClosedAt:                    time.Unix(0, 0).UTC(), // avoid nil-time panic
	}

	zeroPreconditions := TransactionOutput{
		MinAccountSequence:          null.IntFrom(0), // explicitly zero — Valid=true, Int64=0
		MinAccountSequenceAge:       null.IntFrom(0), // explicitly zero
		MinAccountSequenceLedgerGap: null.IntFrom(0), // explicitly zero
		ClosedAt:                    time.Unix(0, 0).UTC(),
	}

	// Confirm the JSON-level structs ARE distinguishable.
	if absentPreconditions.MinAccountSequenceAge.Valid ==
		zeroPreconditions.MinAccountSequenceAge.Valid {
		t.Fatal("precondition: JSON structs should differ in .Valid")
	}

	// 2. Convert both to Parquet.
	parquetAbsent := absentPreconditions.ToParquet().(TransactionOutputParquet)
	parquetZero := zeroPreconditions.ToParquet().(TransactionOutputParquet)

	// 3. Assert: the Parquet outputs should be distinguishable.
	//    If they are NOT, the bug is demonstrated (POC_PASS).
	bugsFound := 0

	if parquetAbsent.MinAccountSequence == parquetZero.MinAccountSequence {
		t.Errorf("MinAccountSequence: absent and explicit-zero both produce %d — indistinguishable in Parquet",
			parquetAbsent.MinAccountSequence)
		bugsFound++
	}
	if parquetAbsent.MinAccountSequenceAge == parquetZero.MinAccountSequenceAge {
		t.Errorf("MinAccountSequenceAge: absent and explicit-zero both produce %d — indistinguishable in Parquet",
			parquetAbsent.MinAccountSequenceAge)
		bugsFound++
	}
	if parquetAbsent.MinAccountSequenceLedgerGap == parquetZero.MinAccountSequenceLedgerGap {
		t.Errorf("MinAccountSequenceLedgerGap: absent and explicit-zero both produce %d — indistinguishable in Parquet",
			parquetAbsent.MinAccountSequenceLedgerGap)
		bugsFound++
	}

	if bugsFound == 0 {
		t.Log("All three precondition fields are distinguishable — no bug found")
	} else {
		t.Logf("Bug demonstrated: %d/3 precondition fields collapse null to zero in Parquet", bugsFound)
	}
}
```

### Test Output

```
=== RUN   TestParquetCollapsesNullPreconditions
    data_integrity_poc_test.go:46: MinAccountSequence: absent and explicit-zero both produce 0 — indistinguishable in Parquet
    data_integrity_poc_test.go:51: MinAccountSequenceAge: absent and explicit-zero both produce 0 — indistinguishable in Parquet
    data_integrity_poc_test.go:56: MinAccountSequenceLedgerGap: absent and explicit-zero both produce 0 — indistinguishable in Parquet
    data_integrity_poc_test.go:64: Bug demonstrated: 3/3 precondition fields collapse null to zero in Parquet
--- FAIL: TestParquetCollapsesNullPreconditions (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.906s
```
