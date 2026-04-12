# 015: `ledger_transaction` drops protocol-23+ post-apply fee changes

**Date**: 2026-04-11
**Severity**: High
**Impact**: Structural data corruption
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformLedgerTransaction` exports protocol-23+ Soroban refund metadata incompletely. The input `ingest.LedgerTransaction` carries non-empty `PostTxApplyFeeChanges`, but `LedgerTransactionOutput` serializes only `FeeChanges` into `tx_fee_meta` and exposes no separate field for the post-apply refund stream, so the raw `ledger_transaction` export silently drops part of the fee-change XDR.

## Root Cause

The transform marshals `transaction.UnsafeMeta` and `transaction.FeeChanges` but never marshals `transaction.PostTxApplyFeeChanges`. The schema compounds that omission because `LedgerTransactionOutput` has no field where post-apply fee changes could be emitted, even though sibling transaction logic already knows that protocol 23+ refunds moved into `PostTxApplyFeeChanges`.

## Reproduction

Create a realistic fee-bump Soroban `ingest.LedgerTransaction` with transaction-meta V4, empty `TxChangesAfter`, non-empty pre-apply `FeeChanges`, and non-empty `PostTxApplyFeeChanges`. Feeding that object through `TransformLedgerTransaction` yields a row whose `tx_fee_meta` decodes back to the pre-apply changes only, while the post-apply refund entries remain absent from every exported field.

## Affected Code

- `internal/transform/ledger_transaction.go:27-35` — marshals `UnsafeMeta` and `FeeChanges`, but never `PostTxApplyFeeChanges`.
- `internal/transform/ledger_transaction.go:47-55` — constructs `LedgerTransactionOutput` without any post-apply fee field.
- `internal/transform/schema.go:86-94` — exported schema has no column for post-apply fee changes.
- `internal/transform/transaction.go:195-202` — sibling transaction transform explicitly appends `PostTxApplyFeeChanges` for protocol-23+ refund accounting, confirming the missing data is semantically required.
- `cmd/export_ledger_transaction.go:33-42` — production export path emits the incomplete transform result for every ledger transaction row.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPostTxApplyFeeChangesDroppedFromLedgerTransaction`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./...`, create `internal/transform/data_integrity_poc_test.go` with the code below, then run `go test ./internal/transform/... -run TestPostTxApplyFeeChangesDroppedFromLedgerTransaction -v`.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestPostTxApplyFeeChangesDroppedFromLedgerTransaction(t *testing.T) {
	feeChanges := xdr.LedgerEntryChanges{
		{
			Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
			State: &xdr.LedgerEntry{
				Data: xdr.LedgerEntryData{
					Type: xdr.LedgerEntryTypeAccount,
					Account: &xdr.AccountEntry{
						AccountId: testAccount5ID,
						Balance:   4067134559286,
					},
				},
			},
		},
		{
			Type: xdr.LedgerEntryChangeTypeLedgerEntryUpdated,
			Updated: &xdr.LedgerEntry{
				Data: xdr.LedgerEntryData{
					Type: xdr.LedgerEntryTypeAccount,
					Account: &xdr.AccountEntry{
						AccountId: testAccount5ID,
						Balance:   4067134520753,
					},
				},
			},
		},
	}
	postApplyChanges := xdr.LedgerEntryChanges{
		{
			Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
			State: &xdr.LedgerEntry{
				Data: xdr.LedgerEntryData{
					Type: xdr.LedgerEntryTypeAccount,
					Account: &xdr.AccountEntry{
						AccountId: testAccount5ID,
						Balance:   4067134520753,
					},
				},
			},
		},
		{
			Type: xdr.LedgerEntryChangeTypeLedgerEntryUpdated,
			Updated: &xdr.LedgerEntry{
				Data: xdr.LedgerEntryData{
					Type: xdr.LedgerEntryTypeAccount,
					Account: &xdr.AccountEntry{
						AccountId: testAccount5ID,
						Balance:   4067134531458,
					},
				},
			},
		},
	}

	tx := ingest.LedgerTransaction{
		Index: 1,
		UnsafeMeta: xdr.TransactionMeta{
			V: 4,
			V4: &xdr.TransactionMetaV4{
				TxChangesBefore: xdr.LedgerEntryChanges{},
				Operations:      []xdr.OperationMetaV2{{}},
				TxChangesAfter:  xdr.LedgerEntryChanges{},
				SorobanMeta: &xdr.SorobanTransactionMetaV2{
					Ext: xdr.SorobanTransactionMetaExt{
						V: 1,
						V1: &xdr.SorobanTransactionMetaExtV1{
							TotalNonRefundableResourceFeeCharged: 20230,
							TotalRefundableResourceFeeCharged:    7298,
							RentFeeCharged:                       7278,
						},
					},
				},
			},
		},
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTxFeeBump,
			FeeBump: &xdr.FeeBumpTransactionEnvelope{
				Tx: xdr.FeeBumpTransaction{
					FeeSource: testAccount5,
					Fee:       38533,
					InnerTx: xdr.FeeBumpTransactionInnerTx{
						Type: xdr.EnvelopeTypeEnvelopeTypeTx,
						V1: &xdr.TransactionV1Envelope{
							Tx: xdr.Transaction{
								SourceAccount: testAccount1,
								SeqNum:        112351890582290871,
								Fee:           38333,
								Operations: []xdr.Operation{
									{
										SourceAccount: &testAccount2,
										Body: xdr.OperationBody{
											Type: xdr.OperationTypeCreateAccount,
											CreateAccountOp: &xdr.CreateAccountOp{
												Destination:     testAccount4ID,
												StartingBalance: 1000000000,
											},
										},
									},
								},
								Ext: xdr.TransactionExt{
									V: 1,
									SorobanData: &xdr.SorobanTransactionData{
										ResourceFee: 38233,
									},
								},
							},
						},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			Result: xdr.TransactionResult{
				FeeCharged: 27828,
				Result: xdr.TransactionResultResult{
					Code: xdr.TransactionResultCodeTxFeeBumpInnerSuccess,
					InnerResultPair: &xdr.InnerTransactionResultPair{
						Result: xdr.InnerTransactionResult{
							FeeCharged: 100,
							Result: xdr.InnerTransactionResultResult{
								Code: xdr.TransactionResultCodeTxSuccess,
								Results: &[]xdr.OperationResult{
									{
										Tr: &xdr.OperationResultTr{
											Type: xdr.OperationTypeCreateAccount,
											CreateAccountResult: &xdr.CreateAccountResult{
												Code: xdr.CreateAccountResultCodeCreateAccountSuccess,
											},
										},
									},
								},
							},
						},
					},
					Results: &[]xdr.OperationResult{{}},
				},
			},
		},
		FeeChanges:            feeChanges,
		PostTxApplyFeeChanges: postApplyChanges,
	}

	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq:     61396807,
			LedgerVersion: 25,
			ScpValue:      xdr.StellarValue{CloseTime: 1700000000},
		},
	}

	output, err := TransformLedgerTransaction(tx, lhe)
	if err != nil {
		t.Fatalf("TransformLedgerTransaction returned error: %v", err)
	}

	if len(tx.PostTxApplyFeeChanges) == 0 {
		t.Fatal("test setup error: PostTxApplyFeeChanges must be non-empty")
	}

	var gotFeeMeta xdr.LedgerEntryChanges
	if err := xdr.SafeUnmarshalBase64(output.TxFeeMeta, &gotFeeMeta); err != nil {
		t.Fatalf("failed to decode tx_fee_meta: %v", err)
	}
	if !reflect.DeepEqual(gotFeeMeta, feeChanges) {
		t.Fatalf("unexpected tx_fee_meta contents: got %#v want %#v", gotFeeMeta, feeChanges)
	}

	combinedFeeChanges := append(append(xdr.LedgerEntryChanges{}, feeChanges...), postApplyChanges...)
	if reflect.DeepEqual(gotFeeMeta, combinedFeeChanges) {
		t.Fatal("hypothesis disproven: tx_fee_meta already includes post-apply fee changes")
	}

	var gotMeta xdr.TransactionMeta
	if err := xdr.SafeUnmarshalBase64(output.TxMeta, &gotMeta); err != nil {
		t.Fatalf("failed to decode tx_meta: %v", err)
	}
	metaV4, ok := gotMeta.GetV4()
	if !ok {
		t.Fatalf("expected tx_meta to decode as v4, got version %d", gotMeta.V)
	}
	if len(metaV4.TxChangesAfter) != 0 {
		t.Fatalf("test setup invalid: expected empty TxChangesAfter, got %d entries", len(metaV4.TxChangesAfter))
	}
	if reflect.DeepEqual(metaV4.TxChangesAfter, postApplyChanges) {
		t.Fatal("hypothesis disproven: tx_meta already carries post-apply fee changes")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Protocol-23+ `ledger_transaction` exports should preserve the full fee-change XDR needed to reconstruct Soroban refunds, either by including `PostTxApplyFeeChanges` in `tx_fee_meta` or by exporting it in a dedicated field.
- **Actual**: `tx_fee_meta` contains only pre-apply `FeeChanges`, `tx_meta` contains V4 metadata with empty `TxChangesAfter`, and no other output field carries the post-apply refund changes.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC builds a real V4 fee-bump Soroban `ingest.LedgerTransaction`, runs `TransformLedgerTransaction`, and decodes the emitted `tx_fee_meta`/`tx_meta` blobs.
2. Realistic preconditions: YES — the repository already has production-style P23+ Soroban fixtures with non-empty `PostTxApplyFeeChanges`, and the export path operates on this exact SDK type.
3. Bug vs by-design: BUG — sibling transaction logic already appends `PostTxApplyFeeChanges` for P23+ refund accounting, and there is no documented contract stating `ledger_transaction` should discard that refund stream.
4. Final severity: High — this is structural loss of fee metadata. The export omits real refund-change XDR, but the final charged fee still exists elsewhere, so it is not the strongest “wrong numeric field” class.
5. In scope: YES — concrete production transform path, reproducible with normal Stellar protocol-23+ fee-bump Soroban transactions.
6. Test correctness: CORRECT — the test uses production transform code, valid XDR shapes, non-empty post-apply changes, and direct round-trip decoding of the exported blobs.
7. Alternative explanations: NONE — `tx_fee_meta` round-trips exactly `FeeChanges`, while `PostTxApplyFeeChanges` remains present on input and absent from the output schema.
8. Novelty: NOVEL

## Suggested Fix

Add a dedicated `post_tx_apply_fee_meta` field to `LedgerTransactionOutput` (and any matching JSON/Parquet schema) or explicitly append `PostTxApplyFeeChanges` to the exported fee metadata for protocol-23+ transactions. Whichever shape is chosen, keep the raw export lossless so downstream consumers can reconstruct the full fee/refund flow from `ledger_transaction` alone.
