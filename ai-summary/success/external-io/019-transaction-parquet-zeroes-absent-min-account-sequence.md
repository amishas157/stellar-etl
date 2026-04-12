# 019: Transaction Parquet zeroes absent min-account-sequence

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` preserves `min_account_sequence` as nullable state: transactions without a `MinSeqNum` precondition export `null`, while transactions that explicitly set `MinSeqNum=0` export `0`. `TransactionOutput.ToParquet()` erases that distinction by reading `.Int64` from `null.Int` into a plain `int64` Parquet column, so both rows become `0`.

This is reachable in normal Stellar usage. Upstream `txnbuild.Preconditions.BuildXDR()` emits a `PRECOND_V2` `MinSeqNum` whenever the pointer is non-nil, even if the value is zero, so an explicit on-chain `MinSeqNum=0` is distinct from an absent pointer and should not be flattened away.

## Root Cause

The JSON/export schema models `min_account_sequence` as `null.Int`, and `TransformTransaction()` only populates it when `transaction.Envelope.MinSeqNum()` returns a non-nil pointer. The Parquet schema removes that nullability by defining the column as plain `int64`, and the converter copies `to.MinAccountSequence.Int64` without checking `Valid`, turning the invalid zero value of `null.Int{}` into a plausible real export value.

## Reproduction

During normal export, a `PRECOND_V2` transaction with `MinSeqNum` absent produces `min_account_sequence=null`, while a `PRECOND_V2` transaction with `MinSeqNum` explicitly set to `0` produces `min_account_sequence=0`. Running the same transformed rows through the Parquet writer rewrites both rows to `min_account_sequence=0`, so downstream consumers lose the distinction between “not set” and “explicit zero”.

## Affected Code

- `internal/transform/transaction.go:113-117` — reads `Envelope.MinSeqNum()` and only sets `outputMinSequence` when the pointer is non-nil.
- `internal/transform/transaction.go:233-254` — preserves that nullable value in `TransactionOutput.MinAccountSequence`.
- `internal/transform/schema.go:65` — JSON/export schema keeps `min_account_sequence` as `null.Int`.
- `internal/transform/parquet_converter.go:59-86` — `TransactionOutput.ToParquet()` flattens `null.Int` by copying `.Int64`.
- `internal/transform/schema_parquet.go:56` — Parquet schema defines `min_account_sequence` as plain `int64`.
- `cmd/export_transactions.go:51-65` — normal `--write-parquet` export path writes these lossy Parquet rows.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTransactionMinAccountSequenceNullCollapseInParquet`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestTransactionMinAccountSequenceNullCollapseInParquet(t *testing.T) {
	hardCodedTransactionHash := xdr.Hash([32]byte{0x01})
	genericResultResults := &[]xdr.OperationResult{
		{
			Tr: &xdr.OperationResultTr{
				Type:                xdr.OperationTypeCreateAccount,
				CreateAccountResult: &xdr.CreateAccountResult{Code: 0},
			},
		},
	}

	meta := xdr.TransactionMeta{
		V:  1,
		V1: genericTxMeta,
	}
	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 100,
		},
	}

	txAbsent := ingest.LedgerTransaction{
		Index:      1,
		UnsafeMeta: meta,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount1,
					SeqNum:        1,
					Fee:           100,
					Cond: xdr.Preconditions{
						Type: xdr.PreconditionTypePrecondV2,
						V2:   &xdr.PreconditionsV2{},
					},
					Operations: []xdr.Operation{
						{
							Body: xdr.OperationBody{
								Type:           xdr.OperationTypeBumpSequence,
								BumpSequenceOp: &xdr.BumpSequenceOp{BumpTo: 1},
							},
						},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: hardCodedTransactionHash,
			Result: xdr.TransactionResult{
				FeeCharged: 100,
				Result: xdr.TransactionResultResult{
					Code:    xdr.TransactionResultCodeTxSuccess,
					Results: genericResultResults,
				},
			},
		},
	}

	explicitZeroSeqNum := xdr.SequenceNumber(0)
	txExplicitZero := ingest.LedgerTransaction{
		Index:      2,
		UnsafeMeta: meta,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount1,
					SeqNum:        1,
					Fee:           100,
					Cond: xdr.Preconditions{
						Type: xdr.PreconditionTypePrecondV2,
						V2: &xdr.PreconditionsV2{
							MinSeqNum: &explicitZeroSeqNum,
						},
					},
					Operations: []xdr.Operation{
						{
							Body: xdr.OperationBody{
								Type:           xdr.OperationTypeBumpSequence,
								BumpSequenceOp: &xdr.BumpSequenceOp{BumpTo: 1},
							},
						},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: hardCodedTransactionHash,
			Result: xdr.TransactionResult{
				FeeCharged: 100,
				Result: xdr.TransactionResultResult{
					Code:    xdr.TransactionResultCodeTxSuccess,
					Results: genericResultResults,
				},
			},
		},
	}

	absentOutput, err := TransformTransaction(txAbsent, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction(absent): %v", err)
	}
	explicitOutput, err := TransformTransaction(txExplicitZero, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction(explicit zero): %v", err)
	}

	if absentOutput.MinAccountSequence.Valid {
		t.Fatalf("expected absent MinAccountSequence to remain null, got valid value %d", absentOutput.MinAccountSequence.Int64)
	}
	if !explicitOutput.MinAccountSequence.Valid || explicitOutput.MinAccountSequence.Int64 != 0 {
		t.Fatalf("expected explicit zero MinAccountSequence to remain valid 0, got %+v", explicitOutput.MinAccountSequence)
	}

	type minSeqJSON struct {
		MinAccountSequence *int64 `json:"min_account_sequence"`
	}

	absentJSON, err := json.Marshal(absentOutput)
	if err != nil {
		t.Fatalf("marshal absent output: %v", err)
	}
	explicitJSON, err := json.Marshal(explicitOutput)
	if err != nil {
		t.Fatalf("marshal explicit output: %v", err)
	}

	var absentParsed, explicitParsed minSeqJSON
	if err := json.Unmarshal(absentJSON, &absentParsed); err != nil {
		t.Fatalf("unmarshal absent output: %v", err)
	}
	if err := json.Unmarshal(explicitJSON, &explicitParsed); err != nil {
		t.Fatalf("unmarshal explicit output: %v", err)
	}

	if absentParsed.MinAccountSequence != nil {
		t.Fatalf("expected JSON null for absent MinAccountSequence, got %d", *absentParsed.MinAccountSequence)
	}
	if explicitParsed.MinAccountSequence == nil || *explicitParsed.MinAccountSequence != 0 {
		t.Fatalf("expected JSON 0 for explicit-zero MinAccountSequence, got %v", explicitParsed.MinAccountSequence)
	}

	absentParquet := absentOutput.ToParquet().(TransactionOutputParquet)
	explicitParquet := explicitOutput.ToParquet().(TransactionOutputParquet)

	if absentParquet.MinAccountSequence == explicitParquet.MinAccountSequence {
		t.Fatalf(
			"MinAccountSequence collapsed in Parquet: absent=%d explicitZero=%d",
			absentParquet.MinAccountSequence,
			explicitParquet.MinAccountSequence,
		)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Parquet should preserve the distinction between an absent `min_account_sequence` and an explicit `min_account_sequence=0`, either via nullable columns or a separate validity marker.
- **Actual**: Parquet rewrites absent `min_account_sequence` to `0`, colliding with real transactions that explicitly set `MinSeqNum=0`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs the full production path through `TransformTransaction()` and then the real `TransactionOutput.ToParquet()` converter.
2. Realistic preconditions: YES — Stellar `PRECOND_V2` transactions can encode an explicit `MinSeqNum=0`; upstream `txnbuild` emits it whenever the pointer is non-nil, even when the value is zero.
3. Bug vs by-design: BUG — the export schema already treats the field as nullable in JSON, so flattening it only in Parquet silently changes the data contract rather than intentionally omitting a field.
4. Final severity: High — the export produces structurally wrong Parquet data that downstream systems can consume without any error.
5. In scope: YES — this is silent corruption in a shipped export format.
6. Test correctness: CORRECT — the rewritten PoC avoids hand-constructed output structs, proves the upstream null-vs-zero distinction first, and then shows the Parquet collision.
7. Alternative explanations: NONE — the collision occurs specifically because `ToParquet()` reads `.Int64` from an invalid `null.Int`.
8. Novelty: NOT ASSESSED — duplicate handling is delegated to the orchestrator.

## Suggested Fix

Make `min_account_sequence` nullable in Parquet, or add a parallel presence/validity column so consumers can distinguish absent values from explicit zero. The converter should not serialize `null.Int{}` as an indistinguishable scalar `0`.
