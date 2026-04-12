# H004: Protocol-20 fee-bump Soroban `fee_charged` drops the inclusion fee

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For any protocol-20 Soroban fee-bump transaction, exported `fee_charged` should equal the full net fee deducted for the transaction: `inclusion_fee_charged + resource_fee - resource_fee_refund`. On a non-refundable fee-bump Soroban transaction, that means `fee_charged` should still include both the outer inclusion fee and the Soroban resource fee.

## Mechanism

`TransformTransaction()` contains a special protocol-20 fee-bump workaround, but the replacement formula is `resource_fee - resource_fee_refund`. That recomputation drops `inclusion_fee_charged` entirely, so every pre-protocol-21 fee-bump Soroban transaction underreports `fee_charged` by at least the inclusion-fee component when no refund occurs, and produces a malformed net-fee value even before any refund-recipient issues are considered.

## Trigger

1. Export a protocol-20 Soroban fee-bump transaction with `LedgerVersion < 21`.
2. Use a case with `resource_fee_refund = 0` (or any non-refundable Soroban fee-bump).
3. Inspect `history_transactions.fee_charged`, `inclusion_fee_charged`, and `resource_fee`.
4. The row will report `fee_charged = resource_fee` instead of `inclusion_fee_charged + resource_fee`.

## Target Code

- `internal/transform/transaction.go:177-180` — derives `inclusion_fee_charged` from the initial fee-account deduction.
- `internal/transform/transaction.go:215-217` — protocol-20 fee-bump workaround overwrites `fee_charged` with `resource_fee - resource_fee_refund`.
- `internal/transform/schema.go:37-44, 77-84` — transaction schema exports `fee_charged`, `inclusion_fee_charged`, `resource_fee`, and `resource_fee_refund` as independent numeric fields.

## Evidence

The function first computes `inclusion_fee_charged` from the pre-apply deduction (`initialFeeCharged - outputResourceFee`), establishing that the inclusion-fee component is available. It then overwrites `fee_charged` for all protocol-20 fee-bump Soroban transactions using only `resource_fee` and `resource_fee_refund`, which mathematically excludes the inclusion fee from the exported total.

## Anti-Evidence

The branch is guarded by an explicit comment referencing stellar-core issue 4188, so the code is clearly attempting to compensate for a real protocol-20 bug. A reviewer may conclude the intended contract was only to correct the refundable Soroban portion, but that still leaves the exported `fee_charged` inconsistent with the surrounding fee breakdown columns and with the net fee actually deducted.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The protocol-20 fee-bump workaround at `transaction.go:215-217` sets `outputFeeCharged = outputResourceFee - outputResourceFeeRefund`, omitting `outputInclusionFeeCharged`. The upstream authoritative `stellar/go` processor at `processors/transaction/transaction.go:234` uses the correct formula: `outputFeeCharged = outputResourceFee - outputResourceFeeRefund + outputInclusionFeeCharged`. This is a confirmed divergence from the upstream source of truth — the local code drops the inclusion fee term, underreporting `fee_charged` for every protocol-20 fee-bump Soroban transaction.

### Code Paths Examined

- `internal/transform/transaction.go:42` — initial `outputFeeCharged = int64(transaction.Result.Result.FeeCharged)` from XDR (known incorrect for P20 fee-bump)
- `internal/transform/transaction.go:162` — `outputResourceFee = int64(sorobanData.ResourceFee)`
- `internal/transform/transaction.go:177-179` — `initialFeeCharged = accountBalanceStart - accountBalanceEnd; outputInclusionFeeCharged = initialFeeCharged - outputResourceFee` — correctly computes inclusion fee
- `internal/transform/transaction.go:215-217` — **BUG**: `outputFeeCharged = outputResourceFee - outputResourceFeeRefund` — drops `outputInclusionFeeCharged`
- `stellar/go processors/transaction/transaction.go:233-234` — **upstream correct**: `outputFeeCharged = outputResourceFee - outputResourceFeeRefund + outputInclusionFeeCharged`
- `internal/transform/transaction_test.go:650` — only test uses `LedgerVersion: 25`, so P20 workaround path has zero test coverage

### Findings

The local stellar-etl diverges from the upstream `stellar/go` transaction processor on this exact line. The upstream formula is:
```go
outputFeeCharged = outputResourceFee - outputResourceFeeRefund + outputInclusionFeeCharged
```
The local formula is:
```go
outputFeeCharged = outputResourceFee - outputResourceFeeRefund
```
The missing `+ outputInclusionFeeCharged` term means every P20 fee-bump Soroban transaction's `fee_charged` export is too low by exactly the inclusion fee amount. This is a financial data corruption bug affecting historical ledger exports for ledger versions 20 through (but not including) 21.

The fix is a one-line change: add `+ outputInclusionFeeCharged` to line 216.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go`
- **Setup**: Create a test case with a fee-bump Soroban transaction where `LedgerVersion = 20`. Set up FeeChanges that produce a known `initialFeeCharged` (e.g., account balance drops from 10000 to 8000 → initialFeeCharged = 2000). Set `sorobanData.ResourceFee = 1500` so `inclusionFeeCharged = 2000 - 1500 = 500`. Set TxChangesAfter to produce `resourceFeeRefund = 200`.
- **Steps**: Call `TransformTransaction()` with the constructed input. Extract `FeeCharged` and `InclusionFeeCharged` from the output.
- **Assertion**: Assert `output.FeeCharged == 1500 - 200 + 500` (= 1800, the correct value). The current buggy code will produce `output.FeeCharged == 1500 - 200` (= 1300), which is 500 less than the correct value — exactly the missing inclusion fee.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestP20FeeBumpFeeChargedDropsInclusionFee"
**Test Language**: Go

### Demonstration

The test constructs a protocol-20 fee-bump Soroban transaction with known fee components: resourceFee=1500, inclusionFeeCharged=500 (derived from FeeChanges balance delta 2000 minus resourceFee), and resourceFeeRefund=200. The correct fee_charged should be 1500 - 200 + 500 = 1800, but the buggy code produces 1300, confirming the inclusion fee is entirely dropped from the exported fee_charged value.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestP20FeeBumpFeeChargedDropsInclusionFee demonstrates that for protocol-20
// fee-bump Soroban transactions, the exported fee_charged field is missing the
// inclusion_fee_charged component, underreporting the total fee by that amount.
func TestP20FeeBumpFeeChargedDropsInclusionFee(t *testing.T) {
	// Fee account: testAccount5 (fee bump source)
	// Source account: testAccount1 (inner tx source)
	//
	// Setup values:
	//   accountBalanceStart = 10000, accountBalanceEnd = 8000
	//   => initialFeeCharged = 10000 - 8000 = 2000
	//   resourceFee = 1500
	//   => inclusionFeeCharged = 2000 - 1500 = 500
	//
	//   TxChangesAfter: refund account balance 8000 -> 8200
	//   => resourceFeeRefund = 200
	//
	//   Correct fee_charged = resourceFee - resourceFeeRefund + inclusionFeeCharged
	//                       = 1500 - 200 + 500 = 1800
	//   Buggy fee_charged   = resourceFee - resourceFeeRefund
	//                       = 1500 - 200 = 1300

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
				// TxChangesAfter: fee account refund of 200 (8000 -> 8200)
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
				SorobanMeta: nil,
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
		// FeeChanges: fee account balance drops from 10000 to 8000 (deducted 2000)
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

	// Protocol 20 ledger header — triggers the P20 fee-bump workaround
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

	const expectedResourceFee int64 = 1500
	const expectedInclusionFeeCharged int64 = 500
	const expectedResourceFeeRefund int64 = 200
	const expectedFeeCharged int64 = expectedResourceFee - expectedResourceFeeRefund + expectedInclusionFeeCharged // 1800

	if output.InclusionFeeCharged != expectedInclusionFeeCharged {
		t.Errorf("InclusionFeeCharged: got %d, want %d", output.InclusionFeeCharged, expectedInclusionFeeCharged)
	}

	if output.ResourceFeeRefund != expectedResourceFeeRefund {
		t.Errorf("ResourceFeeRefund: got %d, want %d", output.ResourceFeeRefund, expectedResourceFeeRefund)
	}

	// THE BUG: fee_charged is missing the inclusionFeeCharged component.
	if output.FeeCharged != expectedFeeCharged {
		t.Errorf("FeeCharged corrupted: got %d, want %d (missing inclusionFeeCharged=%d)",
			output.FeeCharged, expectedFeeCharged, expectedInclusionFeeCharged)
	}
}
```

### Test Output

```
=== RUN   TestP20FeeBumpFeeChargedDropsInclusionFee
    data_integrity_poc_test.go:226: FeeCharged corrupted: got 1300, want 1800 (missing inclusionFeeCharged=500)
--- FAIL: TestP20FeeBumpFeeChargedDropsInclusionFee (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.729s
```
