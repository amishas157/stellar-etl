# H001: Transaction Parquet collapses nullable V2 precondition zeroes into absent values

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_transactions --write-parquet` should preserve the same distinction as `TransactionOutput` for `min_account_sequence_age` and `min_account_sequence_ledger_gap`: transactions without `PRECOND_V2` should export those fields as null/absent, while transactions that explicitly set `PRECOND_V2` with `minSeqAge=0` or `minSeqLedgerGap=0` should export literal `0`.

## Mechanism

`TransformTransaction()` models both fields as `null.Int`, and the tests already assert that an explicit V2 zero survives in JSON. But `TransactionOutputParquet` narrows both columns to plain `int64`, and `TransactionOutput.ToParquet()` writes `.Int64`, so every non-V2 transaction is flattened to `0` too. That makes Parquet unable to distinguish "no V2 precondition" from "explicit zero-valued V2 precondition" for two separate transaction columns.

## Trigger

Run `export_transactions --write-parquet` over a range containing:
1. a classic transaction with `PRECOND_NONE` or `PRECOND_TIME`, and
2. a transaction with `PRECOND_V2` where `minSeqAge=0` and/or `minSeqLedgerGap=0`.

Compare the JSON and Parquet outputs for `min_account_sequence_age` and `min_account_sequence_ledger_gap`.

## Target Code

- `internal/transform/schema.go:64-68` — JSON schema models both fields as nullable `null.Int`
- `internal/transform/transaction.go:119-129` — transformer preserves pointer presence from envelope accessors
- `internal/transform/transaction_test.go:165-168` — existing tests assert explicit zero-valued V2 preconditions
- `internal/transform/schema_parquet.go:55-59` — Parquet schema makes both fields required `int64`
- `internal/transform/parquet_converter.go:83-86` — converter serializes `.Int64`, collapsing invalid nulls to `0`

## Evidence

The repository already treats these fields as nullable at the transform layer, and the test fixture explicitly exercises `null.IntFrom(0)` for both columns. The Parquet path removes that nullability exactly the same way the already-published `min_account_sequence` bug does, but for the sibling V2 age and ledger-gap fields.

## Anti-Evidence

A downstream consumer that only checks whether the precondition is restrictive might treat absent and explicit zero as equivalent. But the ETL's own schema intentionally represents both columns as nullable, so flattening both cases to `0` is still a structural loss of transaction-precondition state.

---

## Review

**Verdict**: VIABLE
**Severity**: Low
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — related to success/data-transform/003 and fail/data-transform/023 but neither covers these specific fields with correct analysis

### Trace Summary

Traced from the XDR SDK helpers `Transaction.MinSeqAge()` and `Transaction.MinSeqLedgerGap()` through the transform layer to Parquet output. The SDK helpers return `nil` for non-V2 transactions (PRECOND_NONE, PRECOND_TIME) and `&value` for V2. The transform code correctly preserves this as `null.Int{}` (Valid=false) vs `null.IntFrom(0)` (Valid=true). JSON marshaling respects this: `null` vs `0`. The Parquet converter reads `.Int64` directly, which is `0` in both cases, collapsing the distinction.

### Code Paths Examined

- `stellar/go xdr/transaction.go:46-55` — `MinSeqAge()` returns nil for non-V2, `&tx.Cond.V2.MinSeqAge` for V2
- `stellar/go xdr/transaction.go:59-68` — `MinSeqLedgerGap()` same pattern, returns nil for non-V2
- `internal/transform/transaction.go:119-129` — preserves nil→`null.Int{}`, non-nil→`null.IntFrom(int64(*val))`
- `internal/transform/schema.go:66-67` — `MinAccountSequenceAge null.Int`, `MinAccountSequenceLedgerGap null.Int`
- `internal/transform/schema_parquet.go:57-58` — both as plain `int64`, no optionality
- `internal/transform/parquet_converter.go:85-86` — `to.MinAccountSequenceAge.Int64` and `to.MinAccountSequenceLedgerGap.Int64`
- `guregu/null int.go:105-110` — `MarshalJSON` returns `"null"` when `!Valid`, `"0"` when `Valid && Int64==0`

### Findings

The bug is mechanically identical to the confirmed success/data-transform/003 finding for `min_account_sequence`. However, the **semantic impact is much lower**:

- For `min_account_sequence`: null means "source sequence must equal tx.seqNum - 1" (strict), while 0 means "any sequence ≥ 0" (permissive). These have different enforcement semantics. → High severity.
- For `min_account_sequence_age`: null means "no age constraint" and 0 means "minimum age = 0 seconds" — semantically equivalent (both mean no restriction). → Low severity.
- For `min_account_sequence_ledger_gap`: null means "no gap constraint" and 0 means "minimum gap = 0 ledgers" — semantically equivalent. → Low severity.

The structural discrepancy is real (JSON outputs `null`, Parquet outputs `0`), but the practical impact is minimal because the collapsed values carry the same protocol-level meaning. A downstream consumer querying `WHERE min_account_sequence_age IS NOT NULL` to identify V2 transactions would get wrong results in Parquet, but that query pattern is better served by other V2-specific fields.

Note: fail/data-transform/023 previously rejected this finding, but its reasoning was flawed — it conflated "MinSeqAge is not optional within V2" (true) with "the transform cannot preserve a null state" (false — the SDK helpers return nil for non-V2 transactions, and the transform does preserve this).

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go` or `internal/transform/data_integrity_poc_test.go` (if it exists from success 003's PoC)
- **Setup**: Use `makeTransactionTestInput()` to get base transactions; create two variants — one with PRECOND_TIME (non-V2) and one with PRECOND_V2 where MinSeqAge=0 and MinSeqLedgerGap=0
- **Steps**: Run both through `TransformTransaction()`, verify JSON-layer outputs differ (null vs 0), then run `ToParquet()` on both
- **Assertion**: Assert that `parquetNonV2.MinAccountSequenceAge` and `parquetV2.MinAccountSequenceAge` are distinguishable (they won't be — this demonstrates the bug). Same for MinAccountSequenceLedgerGap.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestParquetCollapsesV2PreconditionZeroes"
**Test Language**: Go

### Demonstration

The test constructs two transactions: one with PRECOND_TIME (non-V2) and one with PRECOND_V2 where MinSeqAge=0 and MinSeqLedgerGap=0. After `TransformTransaction()`, the JSON layer correctly preserves the distinction — non-V2 produces `null.Int{Valid:false}` while V2 produces `null.IntFrom(0)` (Valid:true, Int64:0). However, after `ToParquet()`, both transactions produce identical `int64(0)` values for `MinAccountSequenceAge` and `MinAccountSequenceLedgerGap`, making them indistinguishable in Parquet output.

### Test Body

```go
func TestParquetCollapsesV2PreconditionZeroes(t *testing.T) {
	hardCodedMemoText := "poc-test"
	hardCodedMeta := xdr.TransactionMeta{V: 1, V1: genericTxMeta}
	genericResultResults := &[]xdr.OperationResult{
		{
			Tr: &xdr.OperationResultTr{
				Type:                xdr.OperationTypeCreateAccount,
				CreateAccountResult: &xdr.CreateAccountResult{Code: 0},
			},
		},
	}

	// Transaction 1: PRECOND_TIME (non-V2) — MinSeqAge/MinSeqLedgerGap should be null
	nonV2Tx := ingest.LedgerTransaction{
		Index:      1,
		UnsafeMeta: hardCodedMeta,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount1,
					SeqNum:        112351890582290871,
					Memo:          xdr.Memo{Type: xdr.MemoTypeMemoText, Text: &hardCodedMemoText},
					Fee:           100,
					Cond: xdr.Preconditions{
						Type: xdr.PreconditionTypePrecondTime,
						TimeBounds: &xdr.TimeBounds{
							MinTime: 0,
							MaxTime: 1594272628,
						},
					},
					Operations: []xdr.Operation{
						{
							Body: xdr.OperationBody{
								Type:           xdr.OperationTypeBumpSequence,
								BumpSequenceOp: &xdr.BumpSequenceOp{},
							},
						},
					},
				},
				Signatures: []xdr.DecoratedSignature{
					{Hint: xdr.SignatureHint{1, 2, 3, 4}, Signature: xdr.Signature{1, 2, 3, 4}},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: xdr.Hash{},
			Result: xdr.TransactionResult{
				FeeCharged: 100,
				Result: xdr.TransactionResultResult{
					Code:    xdr.TransactionResultCodeTxSuccess,
					Results: genericResultResults,
				},
			},
		},
	}

	// Transaction 2: PRECOND_V2 with MinSeqAge=0 and MinSeqLedgerGap=0
	v2Tx := ingest.LedgerTransaction{
		Index:      1,
		UnsafeMeta: hardCodedMeta,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount1,
					SeqNum:        112351890582290872,
					Memo:          xdr.Memo{Type: xdr.MemoTypeMemoText, Text: &hardCodedMemoText},
					Fee:           100,
					Cond: xdr.Preconditions{
						Type: xdr.PreconditionTypePrecondV2,
						V2: &xdr.PreconditionsV2{
							TimeBounds: &xdr.TimeBounds{
								MinTime: 0,
								MaxTime: 1594272628,
							},
							MinSeqAge:      0,
							MinSeqLedgerGap: 0,
						},
					},
					Operations: []xdr.Operation{
						{
							Body: xdr.OperationBody{
								Type:           xdr.OperationTypeBumpSequence,
								BumpSequenceOp: &xdr.BumpSequenceOp{},
							},
						},
					},
				},
				Signatures: []xdr.DecoratedSignature{
					{Hint: xdr.SignatureHint{1, 2, 3, 4}, Signature: xdr.Signature{1, 2, 3, 4}},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: xdr.Hash{0x01},
			Result: xdr.TransactionResult{
				FeeCharged: 100,
				Result: xdr.TransactionResultResult{
					Code:    xdr.TransactionResultCodeTxSuccess,
					Results: genericResultResults,
				},
			},
		},
	}

	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq:     30521818,
			LedgerVersion: 21,
			ScpValue:      xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	// Transform both transactions
	nonV2Result, err := TransformTransaction(nonV2Tx, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction (non-V2) failed: %v", err)
	}
	v2Result, err := TransformTransaction(v2Tx, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction (V2) failed: %v", err)
	}

	// Step 1: Verify the JSON layer correctly distinguishes null from zero
	if nonV2Result.MinAccountSequenceAge != (null.Int{}) {
		t.Errorf("JSON layer: non-V2 MinAccountSequenceAge should be null, got %+v", nonV2Result.MinAccountSequenceAge)
	}
	if nonV2Result.MinAccountSequenceLedgerGap != (null.Int{}) {
		t.Errorf("JSON layer: non-V2 MinAccountSequenceLedgerGap should be null, got %+v", nonV2Result.MinAccountSequenceLedgerGap)
	}

	expectedV2 := null.IntFrom(0)
	if v2Result.MinAccountSequenceAge != expectedV2 {
		t.Errorf("JSON layer: V2 MinAccountSequenceAge should be IntFrom(0), got %+v", v2Result.MinAccountSequenceAge)
	}
	if v2Result.MinAccountSequenceLedgerGap != expectedV2 {
		t.Errorf("JSON layer: V2 MinAccountSequenceLedgerGap should be IntFrom(0), got %+v", v2Result.MinAccountSequenceLedgerGap)
	}

	t.Logf("JSON layer correctly distinguishes: non-V2 Age=%+v, V2 Age=%+v", nonV2Result.MinAccountSequenceAge, v2Result.MinAccountSequenceAge)
	t.Logf("JSON layer correctly distinguishes: non-V2 Gap=%+v, V2 Gap=%+v", nonV2Result.MinAccountSequenceLedgerGap, v2Result.MinAccountSequenceLedgerGap)

	// Step 2: Convert both to Parquet and show that the distinction is lost
	nonV2Parquet := nonV2Result.ToParquet().(TransactionOutputParquet)
	v2Parquet := v2Result.ToParquet().(TransactionOutputParquet)

	t.Logf("Parquet non-V2 MinAccountSequenceAge: %d", nonV2Parquet.MinAccountSequenceAge)
	t.Logf("Parquet V2     MinAccountSequenceAge: %d", v2Parquet.MinAccountSequenceAge)
	t.Logf("Parquet non-V2 MinAccountSequenceLedgerGap: %d", nonV2Parquet.MinAccountSequenceLedgerGap)
	t.Logf("Parquet V2     MinAccountSequenceLedgerGap: %d", v2Parquet.MinAccountSequenceLedgerGap)

	// BUG DEMONSTRATION: Both produce the same int64(0) in Parquet
	if nonV2Parquet.MinAccountSequenceAge == v2Parquet.MinAccountSequenceAge {
		t.Errorf("BUG CONFIRMED: Parquet MinAccountSequenceAge is identical for non-V2 (%d) and V2 (%d) — null/zero distinction lost",
			nonV2Parquet.MinAccountSequenceAge, v2Parquet.MinAccountSequenceAge)
	}

	if nonV2Parquet.MinAccountSequenceLedgerGap == v2Parquet.MinAccountSequenceLedgerGap {
		t.Errorf("BUG CONFIRMED: Parquet MinAccountSequenceLedgerGap is identical for non-V2 (%d) and V2 (%d) — null/zero distinction lost",
			nonV2Parquet.MinAccountSequenceLedgerGap, v2Parquet.MinAccountSequenceLedgerGap)
	}

	if nonV2Parquet.MinAccountSequenceAge != 0 {
		t.Errorf("Expected non-V2 Parquet MinAccountSequenceAge to be 0 (collapsed from null), got %d",
			nonV2Parquet.MinAccountSequenceAge)
	}
	if v2Parquet.MinAccountSequenceAge != 0 {
		t.Errorf("Expected V2 Parquet MinAccountSequenceAge to be 0, got %d",
			v2Parquet.MinAccountSequenceAge)
	}

	t.Logf("BUG CONFIRMED: Parquet collapses null.Int{Valid:false} and null.IntFrom(0) both to int64(0)")
}
```

### Test Output

```
=== RUN   TestParquetCollapsesV2PreconditionZeroes
    data_integrity_poc_test.go:602: JSON layer correctly distinguishes: non-V2 Age={NullInt64:{Int64:0 Valid:false}}, V2 Age={NullInt64:{Int64:0 Valid:true}}
    data_integrity_poc_test.go:603: JSON layer correctly distinguishes: non-V2 Gap={NullInt64:{Int64:0 Valid:false}}, V2 Gap={NullInt64:{Int64:0 Valid:true}}
    data_integrity_poc_test.go:609: Parquet non-V2 MinAccountSequenceAge: 0
    data_integrity_poc_test.go:610: Parquet V2     MinAccountSequenceAge: 0
    data_integrity_poc_test.go:611: Parquet non-V2 MinAccountSequenceLedgerGap: 0
    data_integrity_poc_test.go:612: Parquet V2     MinAccountSequenceLedgerGap: 0
    data_integrity_poc_test.go:618: BUG CONFIRMED: Parquet MinAccountSequenceAge is identical for non-V2 (0) and V2 (0) — null/zero distinction lost
    data_integrity_poc_test.go:623: BUG CONFIRMED: Parquet MinAccountSequenceLedgerGap is identical for non-V2 (0) and V2 (0) — null/zero distinction lost
    data_integrity_poc_test.go:637: BUG CONFIRMED: Parquet collapses null.Int{Valid:false} and null.IntFrom(0) both to int64(0)
--- FAIL: TestParquetCollapsesV2PreconditionZeroes (0.00s)
FAIL
```
