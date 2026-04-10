# 004: Muxed Soroban fee balance mismatch

**Date**: 2026-04-10
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` derives Soroban `inclusion_fee_charged` and `resource_fee_refund` by scanning fee-account balance deltas. When the fee-paying account is muxed, the function compares the muxed `M...` address from the envelope against canonical `G...` account IDs from ledger-entry changes, so the lookup misses and both balances stay zero. That produces silently wrong fee exports, including negative inclusion fees and zero refunds.

## Root Cause

`TransformTransaction()` uses `sourceAccount.Address()` and `feeBumpAccount.Address()` as fee-account lookup keys for Soroban transactions. For muxed accounts those methods return `M...` addresses, but `getAccountBalanceFromLedgerEntryChanges()` only compares against `accountEntry.AccountId.Address()`, which is always the underlying canonical `G...` account ID from ledger state. The transformer already has the correct canonicalization helper in `utils.GetAccountAddressFromMuxedAccount()`, and even uses the canonical account form elsewhere in the same function, so this is an inconsistency rather than intended behavior.

## Reproduction

Create a Soroban transaction whose source account is muxed and whose fee/refund balance changes are keyed by the underlying canonical account ID, as real ledger-entry changes are. Run `TransformTransaction()` on that transaction and inspect the exported Soroban fee fields: the ledger deltas imply a 2000 inclusion fee and a 1500 refund, but the export reports `inclusion_fee_charged = -5000` and `resource_fee_refund = 0`.

## Affected Code

- `internal/transform/transaction.go:TransformTransaction:151-159` — selects the fee-account lookup key with raw `.Address()` on muxed accounts
- `internal/transform/transaction.go:TransformTransaction:177-202` — computes `inclusion_fee_charged` and `resource_fee_refund` from the failed balance lookup
- `internal/transform/transaction.go:getAccountBalanceFromLedgerEntryChanges:306-333` — matches only canonical ledger `AccountId.Address()` values
- `internal/utils/main.go:GetAccountAddressFromMuxedAccount:49-53` — existing canonicalization helper that the Soroban fee path should reuse

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestMuxedSorobanFeeBalanceMismatch`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestMuxedSorobanFeeBalanceMismatch -v`
  4. Observe `InclusionFeeCharged` is `-5000` instead of `2000`, and `ResourceFeeRefund` is `0` instead of `1500`.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestMuxedSorobanFeeBalanceMismatch(t *testing.T) {
	muxedSource := xdr.MuxedAccount{
		Type: xdr.CryptoKeyTypeKeyTypeMuxedEd25519,
		Med25519: &xdr.MuxedAccountMed25519{
			Id:      0xdeadbeef,
			Ed25519: *testAccount1ID.Ed25519,
		},
	}

	var resourceFee xdr.Int64 = 5000
	var feeBalanceStart xdr.Int64 = 10_000_000
	var feeBalanceEnd xdr.Int64 = 9_993_000
	var refundBalanceStart xdr.Int64 = 9_993_000
	var refundBalanceEnd xdr.Int64 = 9_994_500

	hardCodedMemoText := "test"
	hardCodedTransactionHash := xdr.Hash([32]byte{0xa8, 0x7f, 0xef, 0x5e})

	transaction := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: muxedSource,
					SeqNum:        1,
					Fee:           10000,
					Memo: xdr.Memo{
						Type: xdr.MemoTypeMemoText,
						Text: &hardCodedMemoText,
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
							ResourceFee: resourceFee,
							Resources: xdr.SorobanResources{
								Footprint: xdr.LedgerFootprint{
									ReadOnly:  []xdr.LedgerKey{},
									ReadWrite: []xdr.LedgerKey{},
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
			TransactionHash: hardCodedTransactionHash,
			Result: xdr.TransactionResult{
				FeeCharged: 7000,
				Result: xdr.TransactionResultResult{
					Code: xdr.TransactionResultCodeTxSuccess,
					Results: &[]xdr.OperationResult{
						{
							Tr: &xdr.OperationResultTr{
								Type:          xdr.OperationTypeBumpSequence,
								BumpSeqResult: &xdr.BumpSequenceResult{Code: 0},
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
							AccountId: testAccount1ID,
							Balance:   feeBalanceStart,
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
							Balance:   feeBalanceEnd,
						},
					},
				},
			},
		},
		UnsafeMeta: xdr.TransactionMeta{
			V: 3,
			V3: &xdr.TransactionMetaV3{
				TxChangesBefore: xdr.LedgerEntryChanges{},
				Operations:      []xdr.OperationMeta{},
				TxChangesAfter: xdr.LedgerEntryChanges{
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
						State: &xdr.LedgerEntry{
							Data: xdr.LedgerEntryData{
								Type: xdr.LedgerEntryTypeAccount,
								Account: &xdr.AccountEntry{
									AccountId: testAccount1ID,
									Balance:   refundBalanceStart,
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
									Balance:   refundBalanceEnd,
								},
							},
						},
					},
				},
			},
		},
	}

	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq:     50_000_000,
			LedgerVersion: 21,
			ScpValue:      xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	output, err := TransformTransaction(transaction, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction returned error: %v", err)
	}

	expectedInclusionFeeCharged := int64(2000)
	expectedResourceFeeRefund := int64(1500)

	if output.InclusionFeeCharged != expectedInclusionFeeCharged {
		t.Errorf("InclusionFeeCharged corrupted by muxed account mismatch: got %d, want %d",
			output.InclusionFeeCharged, expectedInclusionFeeCharged)
	}

	if output.ResourceFeeRefund != expectedResourceFeeRefund {
		t.Errorf("ResourceFeeRefund corrupted by muxed account mismatch: got %d, want %d",
			output.ResourceFeeRefund, expectedResourceFeeRefund)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Soroban fee exports canonicalize the fee-paying account before matching ledger balance changes, so the example transaction reports `inclusion_fee_charged = 2000` and `resource_fee_refund = 1500`.
- **Actual**: The exporter matches the muxed `M...` address directly, misses the real fee-account ledger entries, and emits `inclusion_fee_charged = -5000` plus `resource_fee_refund = 0`.

## Adversarial Review

1. Exercises claimed bug: YES — the test calls `TransformTransaction()` with real `FeeChanges` and `TxChangesAfter` entries keyed by the canonical account ID, which is exactly the path used to derive the exported Soroban fee fields.
2. Realistic preconditions: YES — muxed source and fee-bump accounts are first-class XDR transaction forms, while ledger account entries remain keyed by canonical `AccountId` values.
3. Bug vs by-design: BUG — the same function already canonicalizes muxed accounts for `account` and `fee_account`, and the shared helper exists specifically for that conversion.
4. Final severity: Critical — the defect silently corrupts exported fee numbers used for financial reconciliation and compliance analysis.
5. In scope: YES — this is a concrete data-corruption path in production transformation code, not an SDK bug or a test-only artifact.
6. Test correctness: CORRECT — the assertions compare exported fee fields against balance deltas implied by the same production ledger-change inputs; the test fails because the lookup key is wrong, not because of circular setup.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Canonicalize the Soroban fee-account lookup key before scanning ledger-entry changes. Replace `sourceAccount.Address()` and `feeBumpAccount.Address()` with `utils.GetAccountAddressFromMuxedAccount(...)` or equivalent `ToAccountId().GetAddress()` logic so the lookup always uses canonical `G...` account IDs.
