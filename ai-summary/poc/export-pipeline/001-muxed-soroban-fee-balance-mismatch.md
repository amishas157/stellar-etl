# H001: Muxed Soroban fee accounts break balance-delta fee exports

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Soroban transactions, `inclusion_fee_charged` and `resource_fee_refund` should be derived from the fee-paying account's actual balance deltas even when the source or fee account is muxed. A muxed `M...` account should still match the underlying ledger `AccountId`, so the exported fee fields should reflect the same non-negative values visible in the fee-account balance changes.

## Mechanism

`TransformTransaction()` uses `MuxedAccount.Address()` to build `feeAccountAddress`, which yields an `M...` string for muxed accounts. `getAccountBalanceFromLedgerEntryChanges()` only compares against `accountEntry.AccountId.Address()`, which is the canonical `G...` account ID stored in ledger-entry changes, so muxed fee payers never match and both start/end balances stay zero. The exported `inclusion_fee_charged` and `resource_fee_refund` then collapse to arithmetic on zero instead of the real balance deltas.

## Trigger

Run `export_transactions` on a Soroban transaction whose source account is muxed, or on a Soroban fee-bump transaction whose fee account is muxed, with a non-zero resource fee or refund.

## Target Code

- `internal/transform/transaction.go:151-159` — fee-account lookup key is taken from `sourceAccount.Address()` / `feeBumpAccount.Address()`
- `internal/transform/transaction.go:177-202` — exported fee fields are computed from `getAccountBalanceFromLedgerEntryChanges(...)`
- `internal/transform/transaction.go:306-333` — balance-delta scan only matches `accountEntry.AccountId.Address()`
- `internal/utils/main.go:49-53` — existing helper already canonicalizes muxed accounts to their underlying `AccountId`

## Evidence

The transformer already uses `GetAccountAddressFromMuxedAccount()` for the exported `account` field, which shows the codebase knows ledger state is keyed by canonical `G...` account IDs. But the Soroban fee-account path bypasses that helper and feeds raw muxed `M...` addresses into a matcher that only ever inspects `AccountId.Address()`. `go doc` for `xdr.MuxedAccount.Address` and `xdr.AccountId.Address` confirms these are different encodings for muxed vs canonical accounts.

## Anti-Evidence

Non-muxed source and fee accounts are unaffected because both sides use `G...` addresses in that case. The bug is therefore narrow to muxed-account Soroban transactions, which may make it easy to miss in ordinary fixtures.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete execution path from `TransformTransaction()` through the Soroban fee extraction logic to `getAccountBalanceFromLedgerEntryChanges()`. Confirmed that `sourceAccount.Address()` (line 154) and `feeBumpAccount.Address()` (line 158) produce `M...`-prefixed strings for muxed accounts via `MuxedAccount.GetAddress()` which calls `strkey.Encode(strkey.VersionByteMuxedAccount, ...)`. The comparison target in `getAccountBalanceFromLedgerEntryChanges()` uses `accountEntry.AccountId.Address()` (lines 318, 327) which always produces `G...`-prefixed strings. The mismatch is confirmed and the zero-balance arithmetic produces wrong financial values. Notably, the same function already correctly canonicalizes muxed accounts elsewhere: line 30 uses `GetAccountAddressFromMuxedAccount()` for the output `account` field, and lines 285-291 use `feeBumpAccount.ToAccountId()` for the `FeeAccount` output field.

### Code Paths Examined

- `internal/transform/transaction.go:29` — `sourceAccount := transaction.Envelope.SourceAccount()` returns `xdr.MuxedAccount`
- `internal/transform/transaction.go:30` — `outputAccount` correctly uses `utils.GetAccountAddressFromMuxedAccount()` to canonicalize
- `internal/transform/transaction.go:154` — `feeAccountAddress = sourceAccount.Address()` — BUG: uses raw `.Address()` which returns `M...` for muxed
- `internal/transform/transaction.go:157-158` — `feeBumpAccount.Address()` — BUG: same issue for fee-bump path
- `internal/transform/transaction.go:177` — `getAccountBalanceFromLedgerEntryChanges(transaction.FeeChanges, feeAccountAddress)` — passes muxed `M...` address
- `internal/transform/transaction.go:306-333` — `getAccountBalanceFromLedgerEntryChanges()` compares against `accountEntry.AccountId.Address()` which is always `G...`
- `internal/transform/transaction.go:285-291` — fee-bump detail section correctly uses `feeBumpAccount.ToAccountId()` — confirms the fix pattern exists
- `stellar/go-stellar-sdk xdr/muxed_account.go:122-150` — `GetAddress()` returns `M...` for `CryptoKeyTypeKeyTypeMuxedEd25519`, `G...` for `CryptoKeyTypeKeyTypeEd25519`
- `internal/utils/main.go:49-53` — `GetAccountAddressFromMuxedAccount()` correctly calls `.ToAccountId()` then `.GetAddress()`

### Findings

The bug is confirmed. When a Soroban transaction has a muxed source account (`CryptoKeyTypeKeyTypeMuxedEd25519`):

1. **`feeAccountAddress`** is set to the `M...` encoding (69-char string)
2. **`getAccountBalanceFromLedgerEntryChanges()`** iterates fee changes looking for `accountEntry.AccountId.Address()` matches — these are always `G...` (56-char string)
3. **No match occurs**, so both `accountBalanceStart` and `accountBalanceEnd` remain zero
4. **`initialFeeCharged`** = 0 − 0 = 0 (should be positive)
5. **`outputInclusionFeeCharged`** = 0 − `outputResourceFee` = **negative** (should be positive or zero)
6. **`outputResourceFeeRefund`** = 0 − 0 = 0 (should reflect actual refund)

This corrupts three financial output fields: `inclusion_fee_charged` becomes a large negative number (the negation of the resource fee), `resource_fee_refund` becomes zero, and the implicit `initialFeeCharged` is zero. Downstream consumers doing fee reconciliation or compliance calculations would see nonsensical negative inclusion fees.

The same function already handles muxed accounts correctly in two other locations (line 30 for `outputAccount`, lines 285-291 for `FeeAccount`), confirming this is an oversight rather than a design choice. Existing tests use `CryptoKeyTypeKeyTypeEd25519` source accounts exclusively, so the muxed path is completely untested.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go`
- **Setup**: Create a `ingest.LedgerTransaction` with:
  - A `CryptoKeyTypeKeyTypeMuxedEd25519` source account (use `xdr.MuxedAccountFromAccountId()` with a known G-address and a memo ID)
  - `EnvelopeTypeEnvelopeTypeTx` envelope with Soroban data (`SorobanTransactionData` in the Tx.Ext)
  - `FeeChanges` containing a state/updated pair for the underlying `G...` account with different balances (e.g., start=10000000, end=9000000)
  - Transaction meta V3 with `TxChangesAfter` containing a balance increase (refund)
- **Steps**: Call `TransformTransaction(transaction, lhe)` with the muxed-account Soroban transaction
- **Assertion**: Assert that `output.InclusionFeeCharged` is positive (not negative) and `output.ResourceFeeRefund` is non-zero. Currently, `InclusionFeeCharged` will equal `-resourceFee` (negative) and `ResourceFeeRefund` will be 0, both incorrect. The fix is to replace `sourceAccount.Address()` and `feeBumpAccount.Address()` with canonicalized addresses via `utils.GetAccountAddressFromMuxedAccount()` or the `.ToAccountId().Address()` pattern already used on line 291.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestMuxedSorobanFeeBalanceMismatch"
**Test Language**: Go

### Demonstration

The test constructs a Soroban transaction with a `CryptoKeyTypeKeyTypeMuxedEd25519` source account (muxed version of testAccount1) and FeeChanges/TxChangesAfter keyed by the canonical `G...` AccountId (testAccount1ID). When `TransformTransaction` is called, `feeAccountAddress` becomes the `M...` encoding which fails to match the `G...` entries in `getAccountBalanceFromLedgerEntryChanges()`. This causes `InclusionFeeCharged` to be `-5000` (negative of the resource fee) instead of the correct `2000`, and `ResourceFeeRefund` to be `0` instead of `1500`.

### Test Body

```go
func TestMuxedSorobanFeeBalanceMismatch(t *testing.T) {
	// Use testAccount1 which is a G... account; create a muxed version of it.
	muxedSource := xdr.MuxedAccount{
		Type: xdr.CryptoKeyTypeKeyTypeMuxedEd25519,
		Med25519: &xdr.MuxedAccountMed25519{
			Id:      0xdeadbeef,
			Ed25519: *testAccount1ID.Ed25519,
		},
	}

	// Soroban resource fee declared in the transaction envelope
	var resourceFee xdr.Int64 = 5000

	// Balance deltas for fee changes:
	//   start=10_000_000, end=9_993_000 → deducted 7000 (resourceFee + 2000 inclusion fee)
	var feeBalanceStart xdr.Int64 = 10_000_000
	var feeBalanceEnd xdr.Int64 = 9_993_000

	// Refund: after tx, balance goes from 9_993_000 to 9_994_500 → refund = 1500
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
								Type:             xdr.OperationTypeBumpSequence,
								BumpSeqResult:    &xdr.BumpSequenceResult{Code: 0},
							},
						},
					},
				},
			},
		},
		// FeeChanges use the canonical G... AccountId (as they do in real ledgers)
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
				// Refund shows up in TxChangesAfter
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
			LedgerSeq:     50000000,
			LedgerVersion: 21,
			ScpValue:      xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	output, err := TransformTransaction(transaction, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction returned error: %v", err)
	}

	// Expected values if the bug did NOT exist:
	//   initialFeeCharged = 10_000_000 - 9_993_000 = 7000
	//   InclusionFeeCharged = 7000 - 5000(resourceFee) = 2000
	//   ResourceFeeRefund = 9_994_500 - 9_993_000 = 1500
	//
	// Actual values WITH the bug (muxed M... vs G... mismatch):
	//   balances are both 0 (no match)
	//   initialFeeCharged = 0 - 0 = 0
	//   InclusionFeeCharged = 0 - 5000 = -5000  ← WRONG (negative!)
	//   ResourceFeeRefund = 0 - 0 = 0            ← WRONG (should be 1500)

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

### Test Output

```
=== RUN   TestMuxedSorobanFeeBalanceMismatch
    data_integrity_poc_test.go:191: InclusionFeeCharged corrupted by muxed account mismatch: got -5000, want 2000
    data_integrity_poc_test.go:196: ResourceFeeRefund corrupted by muxed account mismatch: got 0, want 1500
--- FAIL: TestMuxedSorobanFeeBalanceMismatch (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.812s
```
