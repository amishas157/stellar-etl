# H001: Muxed fee accounts break Soroban fee reconciliation

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Soroban transactions, `inclusion_fee_charged` and `resource_fee_refund` should be computed from the actual fee-paying account's balance changes even when the source account or fee-bump fee source is a muxed `M...` address. The transform should normalize muxed accounts to their underlying `G...` account before matching ledger-entry balance changes, so a valid muxed Soroban fee payer produces the same fee breakdown as the equivalent unmuxed account.

## Mechanism

`TransformTransaction()` stores `feeAccountAddress` using `MuxedAccount.Address()`, which returns an `M...` strkey for muxed accounts, but `getAccountBalanceFromLedgerEntryChanges()` compares that string against `AccountId.Address()` values from ledger entries, which are always unmuxed `G...` account IDs. When the fee payer is muxed, both balance lookups miss, so the function computes `initialFeeCharged = 0`, producing corrupt fee-breakdown fields such as negative `inclusion_fee_charged` and zero `resource_fee_refund`.

## Trigger

Process any protocol-20+ Soroban transaction whose fee-paying account is muxed:

1. A classic Soroban transaction with a muxed source account, or
2. A Soroban fee-bump transaction whose `FeeSource` is muxed.

If the transaction charges a resource fee or later refunds part of it, the exported row will reconcile those amounts against a missing balance delta. For example, a transaction with `resource_fee = 38233` and no matched fee-account balance change will export `inclusion_fee_charged = -38233`.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:148-159` — stores the Soroban fee payer as `sourceAccount.Address()` / `feeBumpAccount.Address()`
- `internal/transform/transaction.go:TransformTransaction:177-202` — computes `inclusion_fee_charged` and `resource_fee_refund` from matched balance changes
- `internal/transform/transaction.go:getAccountBalanceFromLedgerEntryChanges:306-333` — matches balance changes against unmuxed `AccountId.Address()` strings
- `internal/utils/main.go:GetAccountAddressFromMuxedAccount:49-54` — existing helper already converts muxed accounts to the underlying `G...` address
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/muxed_account.go:GetAddress:118-147` — SDK returns `M...` strkeys for `KEY_TYPE_MUXED_ED25519`

## Evidence

The transform already uses `GetAccountAddressFromMuxedAccount()` for the exported `account` field, showing the codebase knows muxed accounts must be normalized for account-level data. But the fee-reconciliation branch bypasses that helper and uses `MuxedAccount.Address()` directly. The downstream balance scan then compares that `M...` string to ledger-entry `AccountId.Address()` output, which can only be `G...`, so no balance change can match for a muxed fee payer.

## Anti-Evidence

Most existing transaction fixtures use unmuxed fee payers, so the happy path continues to produce correct values and existing tests pass. Non-Soroban transactions also avoid this branch entirely, so the corruption is limited to Soroban fee accounting rather than every transaction row.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete Soroban fee reconciliation path in `TransformTransaction()`. At lines 154 and 158, `feeAccountAddress` is set via `MuxedAccount.Address()`, which the SDK confirms returns `M...` strkeys for muxed accounts (SDK `muxed_account.go:136-147`). The downstream `getAccountBalanceFromLedgerEntryChanges()` at line 318 compares this against `AccountId.Address()`, which the SDK confirms always returns `G...` strkeys (`account_id.go:28-35`). The string mismatch means both balance lookups return zero, producing `inclusionFeeCharged = -resourceFee` and `resourceFeeRefund = 0`. The same codebase already normalizes muxed addresses correctly at line 30 (`GetAccountAddressFromMuxedAccount`) and line 286 (`.ToAccountId()`), proving this is an oversight in the fee path.

### Code Paths Examined

- `internal/transform/transaction.go:154` — `feeAccountAddress = sourceAccount.Address()` returns `M...` for muxed source accounts
- `internal/transform/transaction.go:157-158` — `feeAccountAddress = feeBumpAccount.Address()` returns `M...` for muxed fee-bump accounts
- `internal/transform/transaction.go:177` — passes `feeAccountAddress` to `getAccountBalanceFromLedgerEntryChanges` for initial fee charge lookup
- `internal/transform/transaction.go:183,201` — passes same `feeAccountAddress` for resource fee refund lookup (V3 and V4 meta paths)
- `internal/transform/transaction.go:306-333` — `getAccountBalanceFromLedgerEntryChanges` compares `accountEntry.AccountId.Address()` (always `G...`) against `sourceAccountAddress` (could be `M...`)
- `go-stellar-sdk/xdr/muxed_account.go:136-147` — SDK `GetAddress()` encodes with `VersionByteMuxedAccount` for muxed type, producing `M...` strkey
- `go-stellar-sdk/xdr/account_id.go:28-35` — SDK `GetAddress()` encodes with `VersionByteAccountID`, always producing `G...` strkey
- `internal/utils/main.go:50-53` — `GetAccountAddressFromMuxedAccount()` correctly normalizes via `ToAccountId()`, producing `G...`
- `internal/transform/transaction.go:30` — `outputAccount` correctly uses `GetAccountAddressFromMuxedAccount()` (proof the codebase knows the pattern)
- `internal/transform/transaction.go:285-286` — Fee-bump detail section correctly uses `.ToAccountId()` (further proof of the oversight)

### Findings

The bug is confirmed at all three call sites where `feeAccountAddress` is used for balance lookups:
1. **Line 177** (FeeChanges lookup): `initialFeeCharged = 0 - 0 = 0`, so `outputInclusionFeeCharged = 0 - resourceFee = -resourceFee`. This produces a **negative inclusion fee** — a physically impossible value that will corrupt any downstream fee analysis.
2. **Line 183** (V3 TxChangesAfter lookup): `outputResourceFeeRefund = 0 - 0 = 0`. Any actual refund is lost.
3. **Line 201** (V4 TxChangesAfter + PostTxApplyFeeChanges lookup): Same zero result, refund lost.

The fix is to normalize `feeAccountAddress` at lines 154 and 158 by using either `GetAccountAddressFromMuxedAccount()` or `sourceAccount.ToAccountId().Address()` / `feeBumpAccount.ToAccountId().Address()`.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go`
- **Setup**: Create a `LedgerTransaction` with a muxed source account (`CryptoKeyTypeKeyTypeMuxedEd25519`) and valid Soroban data. Populate `FeeChanges` with ledger entry changes containing the underlying `G...` account ID with non-zero balance delta. Set `sorobanData.ResourceFee` to a known value (e.g., 38233).
- **Steps**: Call `TransformTransaction(transaction, lhe)` with the muxed-source Soroban transaction.
- **Assertion**: Assert that `result.InclusionFeeCharged` equals the expected positive value (initial fee charged minus resource fee), NOT the negative value `-38233`. Assert `result.ResourceFeeRefund` equals the expected refund amount, NOT zero. Also test the fee-bump variant by constructing a `FeeBump` envelope with a muxed `FeeBumpAccount`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestMuxedFeeAccountBreaksSorobanFeeReconciliation" and "TestMuxedFeeBumpAccountBreaksSorobanFeeReconciliation"
**Test Language**: Go

### Demonstration

Both tests construct Soroban transactions with muxed fee-paying accounts (M... addresses) and ledger entry changes containing the underlying G... account ID with known balance deltas. When `TransformTransaction()` is called, it stores the M... address as `feeAccountAddress`, which fails to match the G... addresses in ledger entries. This causes `getAccountBalanceFromLedgerEntryChanges()` to return zero for both start and end balances, producing `InclusionFeeCharged = -38233` (a physically impossible negative fee) and `ResourceFeeRefund = 0` (losing the actual 10705 stroop refund). The correct values would be `InclusionFeeCharged = 300` and `ResourceFeeRefund = 10705`.

### Test Body

```go
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

	expectedInclusionFeeCharged := feeDeducted - resourceFee // 300
	expectedResourceFeeRefund := refundAmount                // 10705

	t.Logf("InclusionFeeCharged: got=%d, expected_correct=%d", result.InclusionFeeCharged, expectedInclusionFeeCharged)
	t.Logf("ResourceFeeRefund: got=%d, expected_correct=%d", result.ResourceFeeRefund, expectedResourceFeeRefund)

	// BUG DEMONSTRATION: Because feeAccountAddress is "M..." but ledger entries
	// contain "G..." account IDs, the balance lookup misses entirely.
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

	t.Logf("BUG CONFIRMED: Muxed fee account produces InclusionFeeCharged=%d (negative!) and ResourceFeeRefund=%d (should be %d)",
		result.InclusionFeeCharged, result.ResourceFeeRefund, expectedResourceFeeRefund)
}

// TestMuxedFeeBumpAccountBreaksSorobanFeeReconciliation demonstrates the same
// bug for the fee-bump variant: when FeeBumpAccount is muxed, the fee
// reconciliation fails identically.
func TestMuxedFeeBumpAccountBreaksSorobanFeeReconciliation(t *testing.T) {
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

### Test Output

```
=== RUN   TestMuxedFeeAccountBreaksSorobanFeeReconciliation
    data_integrity_poc_test.go:32: Muxed address: MCEODJVUUVYVFD5KT4TOEDTMXQ76OPFOQC2EMYYMLPXQCUVPOB6XQAAAAAAN5LN6567SC
    data_integrity_poc_test.go:33: G... address:  GCEODJVUUVYVFD5KT4TOEDTMXQ76OPFOQC2EMYYMLPXQCUVPOB6XRWPQ
    data_integrity_poc_test.go:198: InclusionFeeCharged: got=-38233, expected_correct=300
    data_integrity_poc_test.go:199: ResourceFeeRefund: got=0, expected_correct=10705
    data_integrity_poc_test.go:226: BUG CONFIRMED: Muxed fee account produces InclusionFeeCharged=-38233 (negative!) and ResourceFeeRefund=0 (should be 10705)
--- PASS: TestMuxedFeeAccountBreaksSorobanFeeReconciliation (0.00s)
=== RUN   TestMuxedFeeBumpAccountBreaksSorobanFeeReconciliation
    data_integrity_poc_test.go:427: InclusionFeeCharged: got=-38233, expected_correct=300
    data_integrity_poc_test.go:428: ResourceFeeRefund: got=0, expected_correct=10705
    data_integrity_poc_test.go:450: BUG CONFIRMED: Muxed fee-bump account produces InclusionFeeCharged=-38233 (negative!) and ResourceFeeRefund=0 (should be 10705)
--- PASS: TestMuxedFeeBumpAccountBreaksSorobanFeeReconciliation (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	(cached)
```
