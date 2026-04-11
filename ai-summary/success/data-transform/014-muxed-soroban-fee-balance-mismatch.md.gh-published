# 014: Muxed Soroban fee balance mismatch

**Date**: 2026-04-11
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` computes Soroban fee breakdowns by looking up fee-payer balance changes in `FeeChanges` and transaction meta. For muxed fee payers, it stores the fee account as an `M...` address but compares that string against ledger-entry `AccountId.Address()` values, which are always unmuxed `G...` addresses, so the lookup silently misses and exports wrong fee numbers.

This reproduces for both classic Soroban transactions with a muxed source account and fee-bump Soroban transactions with a muxed fee source. In both cases the current code exports `inclusion_fee_charged = -38233` and `resource_fee_refund = 0` for a transaction whose correct values are `300` and `10705`.

## Root Cause

In the Soroban branch, `TransformTransaction()` records the fee payer with `sourceAccount.Address()` or `feeBumpAccount.Address()`. For fully muxed accounts the SDK returns an `M...` strkey, but `getAccountBalanceFromLedgerEntryChanges()` matches against `accountEntry.AccountId.Address()`, which is derived from `AccountId` and therefore always yields the underlying `G...` account. Because the strings can never match for muxed fee payers, both the initial fee debit and the later refund lookup fall back to zero balances.

## Reproduction

During normal operation, this occurs whenever a Soroban transaction uses a muxed source account or a fee-bump transaction uses a muxed fee source. The rest of the transaction can be completely valid: the bug is triggered only by the address encoding mismatch between the envelope fee payer (`MuxedAccount`) and the ledger balance changes (`AccountId`).

The existing unmuxed Soroban fee-bump fixture in `transaction_test.go` already shows the correct baseline for the same fee values (`inclusion_fee_charged = 300`, `resource_fee_refund = 10705`). Changing only the fee payer to a muxed form flips those exported fields to `-38233` and `0`.

## Affected Code

- `internal/transform/transaction.go:TransformTransaction:151-159` — captures the Soroban fee payer with `MuxedAccount.Address()`
- `internal/transform/transaction.go:TransformTransaction:177-184` — derives `inclusion_fee_charged` and V3 refund data from matched fee-account balance deltas
- `internal/transform/transaction.go:TransformTransaction:198-202` — derives V4 refund data from the same string-matched balance scan
- `internal/transform/transaction.go:getAccountBalanceFromLedgerEntryChanges:306-333` — compares ledger-entry `AccountId.Address()` values against the supplied fee-account string

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestMuxedFeeAccountBreaksSorobanFeeReconciliation` and `TestMuxedFeeBumpAccountBreaksSorobanFeeReconciliation`
- **Test language**: go
- **How to run**: Create the target test file with the test body below, then run `go build ./...` and `go test ./internal/transform/... -run 'TestMuxedFee(Account|BumpAccount)BreaksSorobanFeeReconciliation' -v`.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestMuxedFeeAccountBreaksSorobanFeeReconciliation demonstrates that when
// a Soroban transaction's fee-paying account is muxed (M... address),
// the fee reconciliation logic fails to match balance changes (which use
// G... addresses), producing a negative inclusion_fee_charged and zero
// resource_fee_refund.
func TestMuxedFeeAccountBreaksSorobanFeeReconciliation(t *testing.T) {
	// Use testAccount1 as the underlying G... account
	// Create a muxed version of the same account
	muxedSource := xdr.MuxedAccount{
		Type: xdr.CryptoKeyTypeKeyTypeMuxedEd25519,
		Med25519: &xdr.MuxedAccountMed25519{
			Id:      0xdeadbeef,
			Ed25519: *testAccount1ID.Ed25519,
		},
	}

	// Verify the muxed account produces an M... address (different from G...)
	muxedAddr := muxedSource.Address()
	gAddr := testAccount1ID.Address()
	if muxedAddr == gAddr {
		t.Fatalf("Expected muxed address to differ from G... address, got same: %s", muxedAddr)
	}
	t.Logf("Muxed address: %s", muxedAddr)
	t.Logf("G... address:  %s", gAddr)

	var resourceFee int64 = 38233
	var initialBalance int64 = 4_000_000_000_000
	var feeDeducted int64 = 38533 // total fee deducted from account
	var balanceAfterFee int64 = initialBalance - feeDeducted
	var refundAmount int64 = 10705
	var balanceAfterRefund int64 = balanceAfterFee + refundAmount

	hardCodedMemoText := "test"

	tx := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: muxedSource,
					SeqNum:        112351890582290871,
					Memo: xdr.Memo{
						Type: xdr.MemoTypeMemoText,
						Text: &hardCodedMemoText,
					},
					Fee: xdr.Uint32(feeDeducted),
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
					Ext: xdr.TransactionExt{
						V: 1,
						SorobanData: &xdr.SorobanTransactionData{
							ResourceFee: xdr.Int64(resourceFee),
						},
					},
				},
				Signatures: []xdr.DecoratedSignature{
					{
						Hint:      xdr.SignatureHint{1, 2, 3, 4},
						Signature: xdr.Signature{1, 2, 3, 4},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: xdr.Hash{},
			Result: xdr.TransactionResult{
				FeeCharged: xdr.Int64(feeDeducted - refundAmount),
				Result: xdr.TransactionResultResult{
					Code: xdr.TransactionResultCodeTxSuccess,
					Results: &[]xdr.OperationResult{
						{
							Tr: &xdr.OperationResultTr{
								Type: xdr.OperationTypeCreateAccount,
								CreateAccountResult: &xdr.CreateAccountResult{Code: 0},
							},
						},
					},
				},
			},
		},
		// FeeChanges use the underlying G... account ID (as real ledger entries always do)
		FeeChanges: xdr.LedgerEntryChanges{
			{
				Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
				State: &xdr.LedgerEntry{
					Data: xdr.LedgerEntryData{
						Type: xdr.LedgerEntryTypeAccount,
						Account: &xdr.AccountEntry{
							AccountId: testAccount1ID,
							Balance:   xdr.Int64(initialBalance),
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
							AccountId: testAccount1ID,
							Balance:   xdr.Int64(balanceAfterFee),
						},
					},
				},
			},
		},
		UnsafeMeta: xdr.TransactionMeta{
			V: 3,
			V3: &xdr.TransactionMetaV3{
				TxChangesBefore: xdr.LedgerEntryChanges{},
				Operations:      []xdr.OperationMeta{{}},
				// TxChangesAfter: refund balance change using G... account
				TxChangesAfter: xdr.LedgerEntryChanges{
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
						State: &xdr.LedgerEntry{
							Data: xdr.LedgerEntryData{
								Type: xdr.LedgerEntryTypeAccount,
								Account: &xdr.AccountEntry{
									AccountId: testAccount1ID,
									Balance:   xdr.Int64(balanceAfterFee),
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
									AccountId: testAccount1ID,
									Balance:   xdr.Int64(balanceAfterRefund),
								},
							},
						},
					},
				},
				SorobanMeta: &xdr.SorobanTransactionMeta{
					ReturnValue: xdr.ScVal{Type: xdr.ScValTypeScvVoid},
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
	}

	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq:     61396807,
			LedgerVersion: 21,
			ScpValue:      xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	result, err := TransformTransaction(tx, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction failed: %v", err)
	}

	// Expected correct values if muxed accounts were normalized:
	// initialFeeCharged = initialBalance - balanceAfterFee = feeDeducted = 38533
	// inclusionFeeCharged = initialFeeCharged - resourceFee = 38533 - 38233 = 300
	// resourceFeeRefund = balanceAfterRefund - balanceAfterFee = refundAmount = 10705
	expectedInclusionFeeCharged := feeDeducted - resourceFee // 300
	expectedResourceFeeRefund := refundAmount                // 10705

	t.Logf("InclusionFeeCharged: got=%d, expected_correct=%d", result.InclusionFeeCharged, expectedInclusionFeeCharged)
	t.Logf("ResourceFeeRefund: got=%d, expected_correct=%d", result.ResourceFeeRefund, expectedResourceFeeRefund)

	// BUG DEMONSTRATION: Because feeAccountAddress is "M..." but ledger entries
	// contain "G..." account IDs, the balance lookup misses entirely.
	// This causes initialFeeCharged=0, so:
	//   inclusionFeeCharged = 0 - resourceFee = -38233 (negative!)
	//   resourceFeeRefund = 0 (refund lost)
	if result.InclusionFeeCharged == expectedInclusionFeeCharged {
		t.Errorf("BUG NOT DEMONSTRATED: InclusionFeeCharged is correct (%d). "+
			"The muxed account normalization may have been fixed.", result.InclusionFeeCharged)
	}

	if result.InclusionFeeCharged != -resourceFee {
		t.Errorf("Expected InclusionFeeCharged to be -resourceFee (%d) due to muxed address mismatch, got %d",
			-resourceFee, result.InclusionFeeCharged)
	}

	if result.ResourceFeeRefund != 0 {
		t.Errorf("Expected ResourceFeeRefund to be 0 due to muxed address mismatch, got %d",
			result.ResourceFeeRefund)
	}

	// The negative inclusion fee is the smoking gun — fees cannot be negative
	if result.InclusionFeeCharged >= 0 {
		t.Errorf("Expected negative InclusionFeeCharged (impossible value), got %d", result.InclusionFeeCharged)
	}

	t.Logf("BUG CONFIRMED: Muxed fee account produces InclusionFeeCharged=%d (negative!) and ResourceFeeRefund=%d (should be %d)",
		result.InclusionFeeCharged, result.ResourceFeeRefund, expectedResourceFeeRefund)
}

// TestMuxedFeeBumpAccountBreaksSorobanFeeReconciliation demonstrates the same
// bug for the fee-bump variant: when FeeBumpAccount is muxed, the fee
// reconciliation fails identically.
func TestMuxedFeeBumpAccountBreaksSorobanFeeReconciliation(t *testing.T) {
	// Create a muxed version of testAccount5 (used as fee-bump source)
	muxedFeeBump := xdr.MuxedAccount{
		Type: xdr.CryptoKeyTypeKeyTypeMuxedEd25519,
		Med25519: &xdr.MuxedAccountMed25519{
			Id:      0xcafebabe,
			Ed25519: *testAccount5ID.Ed25519,
		},
	}

	var resourceFee int64 = 38233
	var initialBalance int64 = 4_067_134_559_286
	var feeDeducted int64 = 38533
	var balanceAfterFee int64 = initialBalance - feeDeducted
	var refundAmount int64 = 10705
	var balanceAfterRefund int64 = balanceAfterFee + refundAmount

	hardCodedMemoText := "test"

	tx := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTxFeeBump,
			FeeBump: &xdr.FeeBumpTransactionEnvelope{
				Tx: xdr.FeeBumpTransaction{
					FeeSource: muxedFeeBump,
					Fee:       xdr.Int64(feeDeducted),
					InnerTx: xdr.FeeBumpTransactionInnerTx{
						Type: xdr.EnvelopeTypeEnvelopeTypeTx,
						V1: &xdr.TransactionV1Envelope{
							Tx: xdr.Transaction{
								SourceAccount: testAccount1,
								SeqNum:        112351890582290871,
								Memo: xdr.Memo{
									Type: xdr.MemoTypeMemoText,
									Text: &hardCodedMemoText,
								},
								Fee: 38333,
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
								Ext: xdr.TransactionExt{
									V: 1,
									SorobanData: &xdr.SorobanTransactionData{
										ResourceFee: xdr.Int64(resourceFee),
									},
								},
							},
						},
					},
				},
				Signatures: []xdr.DecoratedSignature{
					{
						Hint:      xdr.SignatureHint{1, 2, 3, 4},
						Signature: xdr.Signature{1, 2, 3, 4},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: xdr.Hash{},
			Result: xdr.TransactionResult{
				FeeCharged: xdr.Int64(feeDeducted - refundAmount),
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
											Type: xdr.OperationTypeCreateAccount,
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
								Type: xdr.OperationTypeCreateAccount,
								CreateAccountResult: &xdr.CreateAccountResult{Code: 0},
							},
						},
					},
				},
			},
		},
		// FeeChanges use the underlying G... account ID
		FeeChanges: xdr.LedgerEntryChanges{
			{
				Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
				State: &xdr.LedgerEntry{
					Data: xdr.LedgerEntryData{
						Type: xdr.LedgerEntryTypeAccount,
						Account: &xdr.AccountEntry{
							AccountId: testAccount5ID,
							Balance:   xdr.Int64(initialBalance),
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
							Balance:   xdr.Int64(balanceAfterFee),
						},
					},
				},
			},
		},
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
									Balance:   xdr.Int64(balanceAfterFee),
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
									Balance:   xdr.Int64(balanceAfterRefund),
								},
							},
						},
					},
				},
				SorobanMeta: &xdr.SorobanTransactionMeta{
					ReturnValue: xdr.ScVal{Type: xdr.ScValTypeScvVoid},
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
	}

	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq:     61396807,
			LedgerVersion: 21,
			ScpValue:      xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	result, err := TransformTransaction(tx, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction failed: %v", err)
	}

	expectedInclusionFeeCharged := feeDeducted - resourceFee // 300
	expectedResourceFeeRefund := refundAmount                // 10705

	t.Logf("InclusionFeeCharged: got=%d, expected_correct=%d", result.InclusionFeeCharged, expectedInclusionFeeCharged)
	t.Logf("ResourceFeeRefund: got=%d, expected_correct=%d", result.ResourceFeeRefund, expectedResourceFeeRefund)

	// Same bug: muxed fee-bump account M... address won't match G... in ledger entries
	if result.InclusionFeeCharged == expectedInclusionFeeCharged {
		t.Errorf("BUG NOT DEMONSTRATED: InclusionFeeCharged is correct (%d). "+
			"The muxed account normalization may have been fixed.", result.InclusionFeeCharged)
	}

	if result.InclusionFeeCharged != -resourceFee {
		t.Errorf("Expected InclusionFeeCharged to be -resourceFee (%d) due to muxed address mismatch, got %d",
			-resourceFee, result.InclusionFeeCharged)
	}

	if result.ResourceFeeRefund != 0 {
		t.Errorf("Expected ResourceFeeRefund to be 0 due to muxed address mismatch, got %d",
			result.ResourceFeeRefund)
	}

	if result.InclusionFeeCharged >= 0 {
		t.Errorf("Expected negative InclusionFeeCharged (impossible value), got %d", result.InclusionFeeCharged)
	}

	t.Logf("BUG CONFIRMED: Muxed fee-bump account produces InclusionFeeCharged=%d (negative!) and ResourceFeeRefund=%d (should be %d)",
		result.InclusionFeeCharged, result.ResourceFeeRefund, expectedResourceFeeRefund)
}
```

## Expected vs Actual Behavior

- **Expected**: Fee reconciliation should normalize muxed fee payers to their underlying `G...` account before matching ledger-entry balance changes, yielding the same fee breakdown as the unmuxed transaction (`inclusion_fee_charged = 300`, `resource_fee_refund = 10705` in the reproduced cases).
- **Actual**: The code matches an `M...` fee payer against `G...` ledger entries, so both balance scans miss and export `inclusion_fee_charged = -38233` and `resource_fee_refund = 0`.

## Adversarial Review

1. Exercises claimed bug: YES — both tests call the production `TransformTransaction()` path and hit the exact Soroban fee-reconciliation logic that uses `feeAccountAddress`.
2. Realistic preconditions: YES — Stellar XDR uses `MuxedAccount` for both `Transaction.SourceAccount` and `FeeBumpTransaction.FeeSource`, while ledger account entries are keyed by `AccountId`.
3. Bug vs by-design: BUG — the same codebase already normalizes muxed accounts for the exported `account` field and fee-bump detail fields, so the raw `Address()` use in fee reconciliation is an inconsistent oversight rather than intended semantics.
4. Final severity: Critical — the exported fee fields are numeric monetary values consumed by downstream reconciliation and can become negative or zero when they should be positive.
5. In scope: YES — this is a concrete production code path that silently exports corrupt transaction fee data.
6. Test correctness: CORRECT — the PoC constructs real transaction envelopes, uses production ledger-entry change structures, and demonstrates that changing only the fee-payer encoding changes the exported values.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Normalize `feeAccountAddress` before the balance scans. In `TransformTransaction()`, use `utils.GetAccountAddressFromMuxedAccount(sourceAccount)` / `utils.GetAccountAddressFromMuxedAccount(feeBumpAccount)` or `ToAccountId().Address()` instead of raw `MuxedAccount.Address()`.
