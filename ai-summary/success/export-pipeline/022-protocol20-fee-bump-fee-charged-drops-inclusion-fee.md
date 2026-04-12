# 022: Protocol-20 fee-bump fee_charged drops inclusion fee

**Date**: 2026-04-12
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` has a protocol-20 fee-bump Soroban workaround that recomputes `fee_charged`, but the replacement formula omits `inclusion_fee_charged`. As a result, every exported protocol-20 fee-bump Soroban row underreports the total fee by the inclusion-fee component, even when the other fee breakdown fields are correct.

## Root Cause

The transform first derives `outputInclusionFeeCharged` from the fee-account balance delta in `transaction.FeeChanges`, and it separately derives `outputResourceFeeRefund` from `TxChangesAfter`. It then overwrites `outputFeeCharged` for all `ledger_version < 21` fee-bump Soroban transactions with `outputResourceFee - outputResourceFeeRefund`, dropping the already-computed inclusion fee instead of adding it back into the total.

## Reproduction

This manifests during normal export of protocol-20 fee-bump Soroban transactions. In the reproduced case, the fee-account balance delta shows an initial 2000-stroop deduction, Soroban resource fee is 1500, and the refund is 200, so the correct exported total is `1500 - 200 + 500 = 1800`; the current transform emits `1300`.

## Affected Code

- `internal/transform/transaction.go:177-180` — computes `inclusion_fee_charged` from the fee-account balance delta
- `internal/transform/transaction.go:183-184` — computes `resource_fee_refund` from `TxChangesAfter`
- `internal/transform/transaction.go:215-217` — protocol-20 fee-bump workaround overwrites `fee_charged` with a formula that omits `inclusion_fee_charged`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestP20FeeBumpFeeChargedDropsInclusionFee`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestP20FeeBumpFeeChargedDropsInclusionFee(t *testing.T) {
	hardCodedMemoText := "test"
	destination := xdr.MuxedAccount{
		Type:    xdr.CryptoKeyTypeKeyTypeEd25519,
		Ed25519: &xdr.Uint256{1, 2, 3},
	}

	transaction := ingest.LedgerTransaction{
		Index: 1,
		UnsafeMeta: xdr.TransactionMeta{
			V: 3,
			V3: &xdr.TransactionMetaV3{
				TxChangesBefore: xdr.LedgerEntryChanges{},
				Operations:      []xdr.OperationMeta{{}},
				TxChangesAfter: xdr.LedgerEntryChanges{
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
						State: &xdr.LedgerEntry{
							Data: xdr.LedgerEntryData{
								Type: xdr.LedgerEntryTypeAccount,
								Account: &xdr.AccountEntry{
									AccountId: testAccount5ID,
									Balance:   8000,
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
									Balance:   8200,
								},
							},
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
					Fee:       3000,
					InnerTx: xdr.FeeBumpTransactionInnerTx{
						Type: xdr.EnvelopeTypeEnvelopeTypeTx,
						V1: &xdr.TransactionV1Envelope{
							Tx: xdr.Transaction{
								SourceAccount: testAccount1,
								SeqNum:        100,
								Memo: xdr.Memo{
									Type: xdr.MemoTypeMemoText,
									Text: &hardCodedMemoText,
								},
								Fee: 2000,
								Cond: xdr.Preconditions{
									Type: xdr.PreconditionTypePrecondTime,
									TimeBounds: &xdr.TimeBounds{
										MinTime: 0,
										MaxTime: 1700000000,
									},
								},
								Operations: []xdr.Operation{
									{
										SourceAccount: &testAccount2,
										Body: xdr.OperationBody{
											Type: xdr.OperationTypePathPaymentStrictReceive,
											PathPaymentStrictReceiveOp: &xdr.PathPaymentStrictReceiveOp{
												Destination: destination,
											},
										},
									},
								},
								Ext: xdr.TransactionExt{
									V: 1,
									SorobanData: &xdr.SorobanTransactionData{
										ResourceFee: 1500,
									},
								},
							},
						},
					},
				},
				Signatures: []xdr.DecoratedSignature{
					{
						Hint:      xdr.SignatureHint{1, 2, 3, 4},
						Signature: xdr.Signature{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: xdr.Hash{},
			Result: xdr.TransactionResult{
				FeeCharged: 9999,
				Result: xdr.TransactionResultResult{
					Code: xdr.TransactionResultCodeTxFeeBumpInnerSuccess,
					InnerResultPair: &xdr.InnerTransactionResultPair{
						TransactionHash: xdr.Hash{},
						Result: xdr.InnerTransactionResult{
							FeeCharged: 100,
							Result: xdr.InnerTransactionResultResult{
								Code: xdr.TransactionResultCodeTxSuccess,
								Results: &[]xdr.OperationResult{
									{
										Tr: &xdr.OperationResultTr{
											Type:                xdr.OperationTypeCreateAccount,
											CreateAccountResult: &xdr.CreateAccountResult{Code: 0},
										},
									},
								},
							},
						},
					},
					Results: &[]xdr.OperationResult{
						{
							Tr: &xdr.OperationResultTr{
								Type:                xdr.OperationTypeCreateAccount,
								CreateAccountResult: &xdr.CreateAccountResult{Code: 0},
							},
						},
					},
				},
			},
		},
		FeeChanges: xdr.LedgerEntryChanges{
			{
				Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
				State: &xdr.LedgerEntry{
					Data: xdr.LedgerEntryData{
						Type: xdr.LedgerEntryTypeAccount,
						Account: &xdr.AccountEntry{
							AccountId: testAccount5ID,
							Balance:   10000,
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
							Balance:   8000,
						},
					},
				},
			},
		},
	}

	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq:     1000,
			LedgerVersion: 20,
			ScpValue:      xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	output, err := TransformTransaction(transaction, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction failed: %v", err)
	}

	const expectedInclusionFeeCharged int64 = 500
	const expectedResourceFeeRefund int64 = 200
	const expectedFeeCharged int64 = 1800

	if output.InclusionFeeCharged != expectedInclusionFeeCharged {
		t.Fatalf("InclusionFeeCharged: got %d, want %d", output.InclusionFeeCharged, expectedInclusionFeeCharged)
	}

	if output.ResourceFeeRefund != expectedResourceFeeRefund {
		t.Fatalf("ResourceFeeRefund: got %d, want %d", output.ResourceFeeRefund, expectedResourceFeeRefund)
	}

	if output.FeeCharged != expectedFeeCharged {
		t.Fatalf("FeeCharged corrupted: got %d, want %d", output.FeeCharged, expectedFeeCharged)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: For protocol-20 fee-bump Soroban transactions, exported `fee_charged` should equal `resource_fee - resource_fee_refund + inclusion_fee_charged`, matching the total fee actually deducted.
- **Actual**: The export recomputes `fee_charged` as `resource_fee - resource_fee_refund`, underreporting the total by exactly `inclusion_fee_charged`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses the production `TransformTransaction()` path and constructs fee-account ledger changes that independently produce `inclusion_fee_charged=500` and `resource_fee_refund=200` before the faulty overwrite.
2. Realistic preconditions: YES — the code explicitly supports Soroban data on fee-bump envelopes and contains a dedicated `ledger_version < 21` workaround for historical protocol-20 fee-bump transactions, so exporting this path is normal supported behavior.
3. Bug vs by-design: BUG — the surrounding schema exports `fee_charged`, `inclusion_fee_charged`, and `resource_fee_refund` as distinct numeric fields, and the upstream Stellar processor computes the protocol-20 correction as `resource_fee - resource_fee_refund + inclusion_fee_charged`; omitting the inclusion fee here is a local regression, not an intentional contract.
4. Final severity: Critical — `fee_charged` is a monetary field consumed for fee accounting and reconciliation, and the export writes a silently wrong numeric value.
5. In scope: YES — this is concrete financial data corruption in the ETL output, not a theoretical or upstream-only issue.
6. Test correctness: CORRECT — the test does not assert on values it injects directly into the output; it validates that the transform computes two fee components correctly and only the total is wrong after the protocol-20 branch runs.
7. Alternative explanations: NONE — the observed 500-stroop gap is exactly the separately computed `inclusion_fee_charged`, and the same formula in the upstream processor includes that term.
8. Novelty: NOVEL

## Suggested Fix

Change the protocol-20 fee-bump override in `TransformTransaction()` to include `outputInclusionFeeCharged` when recomputing `outputFeeCharged`, matching the upstream processor formula.
