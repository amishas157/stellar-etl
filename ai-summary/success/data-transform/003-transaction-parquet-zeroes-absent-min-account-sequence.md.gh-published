# 003: Transaction Parquet zeroes absent `min_account_sequence`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` preserves whether `min_account_sequence` is absent or explicitly set to `0` by using `null.Int`. The Parquet path drops that distinction by converting the nullable field to a required `int64`, so transactions with no `min_account_sequence` export the same Parquet value as transactions that explicitly set `min_account_sequence=0`.

This is a real structural corruption bug for `min_account_sequence`: under Stellar's precondition semantics, an absent `minSeqNum` means the source account sequence must equal `tx.seqNum - 1`, while an explicit `minSeqNum=0` allows any source sequence in `[0, tx.seqNum)`. The original hypothesis also mentioned the age and ledger-gap fields, but the materially confirmed case is `min_account_sequence`.

## Root Cause

The JSON transaction schema uses `null.Int` for `MinAccountSequence`, and `TransformTransaction()` only sets it when `transaction.Envelope.MinSeqNum()` returns a non-nil pointer. `TransactionOutput.ToParquet()` then ignores the nullable wrapper and writes `to.MinAccountSequence.Int64` into a non-optional Parquet `int64` column, which turns both `null.Int{}` and `null.IntFrom(0)` into the same exported value `0`.

## Reproduction

Create two valid V2-precondition transactions that differ only in `minSeqNum`: one omits it entirely, and one explicitly sets it to `0`. Run both through `TransformTransaction()`, confirm the JSON-layer outputs differ (`null` versus `0`), then convert them with `ToParquet()`. Both Parquet rows contain `min_account_sequence=0`, so the exporter loses the distinction.

## Affected Code

- `internal/transform/transaction.go:113-117` — preserves `min_account_sequence` as `null.Int` and only populates it when the XDR field is present.
- `internal/transform/schema.go:65` — models `min_account_sequence` as nullable in the JSON output schema.
- `internal/transform/schema_parquet.go:56` — defines `min_account_sequence` as a required `int64` in the Parquet schema.
- `internal/transform/parquet_converter.go:84` — serializes `to.MinAccountSequence.Int64` without checking whether the nullable value is valid.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestParquetCollapsesNullPreconditions`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go test ./internal/transform/... -run TestParquetCollapsesNullPreconditions -v`.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestParquetCollapsesNullPreconditions(t *testing.T) {
	transactions, headers, err := makeTransactionTestInput()
	if err != nil {
		t.Fatalf("makeTransactionTestInput() error = %v", err)
	}

	absentMinSeqTx := transactions[0]
	explicitZeroMinSeqTx := transactions[0]
	absentV1 := *transactions[0].Envelope.V1
	explicitZeroV1 := *transactions[0].Envelope.V1
	absentMinSeqTx.Envelope.V1 = &absentV1
	explicitZeroMinSeqTx.Envelope.V1 = &explicitZeroV1

	timeBounds := absentMinSeqTx.Envelope.V1.Tx.Cond.TimeBounds
	explicitZero := xdr.SequenceNumber(0)

	absentMinSeqTx.Envelope.V1.Tx.Cond = xdr.Preconditions{
		Type: xdr.PreconditionTypePrecondV2,
		V2: &xdr.PreconditionsV2{
			TimeBounds: timeBounds,
		},
	}
	explicitZeroMinSeqTx.Envelope.V1.Tx.Cond = xdr.Preconditions{
		Type: xdr.PreconditionTypePrecondV2,
		V2: &xdr.PreconditionsV2{
			TimeBounds: timeBounds,
			MinSeqNum:  &explicitZero,
		},
	}

	absentOutput, err := TransformTransaction(absentMinSeqTx, headers[0])
	if err != nil {
		t.Fatalf("TransformTransaction(absent) error = %v", err)
	}
	explicitZeroOutput, err := TransformTransaction(explicitZeroMinSeqTx, headers[0])
	if err != nil {
		t.Fatalf("TransformTransaction(explicit zero) error = %v", err)
	}

	if absentOutput.MinAccountSequence.Valid {
		t.Fatal("expected absent transaction to keep min_account_sequence null at JSON layer")
	}
	if !explicitZeroOutput.MinAccountSequence.Valid || explicitZeroOutput.MinAccountSequence.Int64 != 0 {
		t.Fatalf("expected explicit-zero transaction to preserve min_account_sequence=0 at JSON layer, got %+v", explicitZeroOutput.MinAccountSequence)
	}

	parquetAbsent := absentOutput.ToParquet().(TransactionOutputParquet)
	parquetExplicitZero := explicitZeroOutput.ToParquet().(TransactionOutputParquet)

	if parquetAbsent.MinAccountSequence == parquetExplicitZero.MinAccountSequence {
		t.Fatalf("parquet collapsed absent min_account_sequence and explicit min_account_sequence=0 to the same value %d", parquetAbsent.MinAccountSequence)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: A transaction with no `min_account_sequence` should remain nullable in Parquet, so it stays distinguishable from a transaction that explicitly sets `min_account_sequence=0`.
- **Actual**: Both states export as the same Parquet value `0`.

## Adversarial Review

1. Exercises claimed bug: YES — the test uses real XDR transaction envelopes, runs `TransformTransaction()`, then runs the production `ToParquet()` converter.
2. Realistic preconditions: YES — V2 preconditions legitimately allow `minSeqNum` to be omitted or explicitly set to `0`.
3. Bug vs by-design: BUG — the ETL already models the field as nullable in its JSON schema, so the Parquet path alone is destroying information.
4. Final severity: High — this is structural corruption of a transaction precondition field that changes the meaning of exported transaction validity constraints.
5. In scope: YES — it is a concrete code path in the transform layer that silently exports wrong data.
6. Test correctness: CORRECT — the test proves the JSON-layer states differ before conversion and that only the Parquet layer collapses them.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Make `TransactionOutputParquet.MinAccountSequence` nullable, such as `*int64` with `repetitiontype=OPTIONAL`, and update `TransactionOutput.ToParquet()` to emit `nil` when `to.MinAccountSequence.Valid` is false and `&value` otherwise. Review the same pattern for other nullable Parquet transaction fields that currently read `.Int64` directly.
