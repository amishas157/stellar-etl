# H003: `ledger_transaction` drops P23+ `PostTxApplyFeeChanges` from all raw blobs

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a protocol-23+ Soroban transaction with a fee refund, the raw
`ledger_transaction` export should preserve every fee-account mutation needed to
replay the transaction's final fee lifecycle. A consumer decoding the row's raw
XDR should be able to recover the post-apply refund changes as well as the initial
fee debit.

## Mechanism

`TransformLedgerTransaction()` base64-encodes `transaction.UnsafeMeta` into
`tx_meta` and `transaction.FeeChanges` into `tx_fee_meta`, but never serializes
`transaction.PostTxApplyFeeChanges`. Because protocol 23 moved refund credits out
of transaction meta and into that second slice, the raw ledger-transaction table
now loses refund-era fee XDR entirely on P23+ ledgers.

## Trigger

Run `export_ledger_transaction` on a protocol-23+ Soroban ledger where a
transaction receives a non-zero resource-fee refund. Decode the row's `tx_meta`
and `tx_fee_meta`: the refund credit will not appear in either blob because
`PostTxApplyFeeChanges` is not exported anywhere.

## Target Code

- `internal/transform/ledger_transaction.go:TransformLedgerTransaction:17-35` — serializes `tx_meta` and `tx_fee_meta` but never touches `PostTxApplyFeeChanges`
- `internal/transform/schema.go:LedgerTransactionOutput:86-94` — `LedgerTransactionOutput` has no field for post-apply fee-change XDR
- `internal/transform/transaction.go:TransformTransaction:195-202` — sibling transaction transform documents that P23+ refunds moved to `PostTxApplyFeeChanges`

## Evidence

The raw ledger-transaction exporter is even more direct than the history
transaction path: it emits only `UnsafeMeta`, `FeeChanges`, and `LedgerHeader`
history. Once P23 moved fee refunds into `PostTxApplyFeeChanges`, this table lost
the only raw blob that could preserve those changes, and there is no alternate
column carrying them.

## Anti-Evidence

As with `history_transactions`, the existing `tx_fee_meta` field is intentionally
the pre-apply debit blob, so the corruption is an omission rather than a bad
reinterpretation of that field. The right fix is probably to add a new raw column,
not to redefine `tx_fee_meta`.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Fail entry 054+055 was about redefining `tx_fee_meta` semantics; this hypothesis correctly targets the absence of a dedicated column, which is the exact reframing 054+055's lesson recommended.

### Trace Summary

Traced the complete data flow from XDR protocol definition through SDK reader to the transform function. `TransactionResultMetaV1` (used in `LedgerCloseMetaV2.TxProcessing`) has `PostTxApplyFeeProcessing` as a field structurally separate from `TxApplyProcessing` (which maps to `UnsafeMeta`). The SDK's `LedgerTransactionReader.Read()` populates `LedgerTransaction.PostTxApplyFeeChanges` from `lcmV2.TxProcessing[i].PostTxApplyFeeProcessing`. `TransformLedgerTransaction()` serializes `UnsafeMeta` → `tx_meta` and `FeeChanges` → `tx_fee_meta` but never reads `PostTxApplyFeeChanges`. Pre-P23, fee refund changes lived in `UnsafeMeta.V3.TxChangesAfter` (part of `tx_meta`); P23+ moved them to `PostTxApplyFeeChanges` (not part of `UnsafeMeta`), creating a silent data regression.

### Code Paths Examined

- `internal/transform/ledger_transaction.go:13-58` — `TransformLedgerTransaction()` serializes `transaction.UnsafeMeta` (line 27) and `transaction.FeeChanges` (line 32) but has no reference to `PostTxApplyFeeChanges`
- `internal/transform/schema.go:86-94` — `LedgerTransactionOutput` struct has fields for `TxMeta`, `TxFeeMeta`, `TxEnvelope`, `TxResult`, `TxLedgerHistory`, `ClosedAt` — no field for post-apply fee changes
- `internal/transform/transaction.go:195-202` — Sibling `TransformTransaction()` explicitly handles P23+ by appending `transaction.PostTxApplyFeeChanges` to `metav4.TxChangesAfter` for refund computation, proving codebase awareness of this field
- `go-stellar-sdk/xdr/xdr_generated.go:18988-19007` — `TransactionResultMetaV1` defines `PostTxApplyFeeProcessing LedgerEntryChanges` as structurally separate from `TxApplyProcessing TransactionMeta`
- `go-stellar-sdk/xdr/xdr_generated.go:18245-18272` — `TransactionMetaV4` contains `TxChangesAfter` but NOT `PostTxApplyFeeProcessing`; the latter is at the `TransactionResultMetaV1` level, not within `TransactionMeta`
- `go-stellar-sdk/ingest/ledger_transaction_reader.go:95-101` — Reader extracts `PostTxApplyFeeChanges` from `lcmV2.TxProcessing[i].PostTxApplyFeeProcessing`, a separate field from `TxApplyProcessing` (which becomes `UnsafeMeta`)

### Findings

1. **Data regression confirmed**: Pre-P23, fee refund balance changes were embedded in `TransactionMeta.V3.TxChangesAfter`, which is part of `UnsafeMeta` and serialized into `tx_meta`. P23+ moved these changes to `PostTxApplyFeeProcessing`, a field at the `TransactionResultMetaV1` level — structurally outside `TransactionMeta`. The `tx_meta` blob no longer contains refund changes for P23+ ledgers.

2. **No alternate export path exists**: `LedgerTransactionOutput` has no field for post-apply fee changes. The schema was not updated for P23. The data is available in the SDK's `LedgerTransaction.PostTxApplyFeeChanges` field but is silently dropped.

3. **Sibling was updated, this was not**: `TransformTransaction()` (lines 195-200) explicitly handles `PostTxApplyFeeChanges` by appending it to `TxChangesAfter` for resource-fee-refund computation. `TransformLedgerTransaction()` was not similarly updated, confirming this is an oversight rather than a design choice.

4. **Impact**: Downstream consumers of the `ledger_transaction` raw export cannot reconstruct fee refund amounts from the exported blobs for P23+ Soroban transactions with non-zero refunds. The exported data looks valid but is silently incomplete.

### PoC Guidance

- **Test file**: `internal/transform/ledger_transaction_test.go`
- **Setup**: Construct an `ingest.LedgerTransaction` with `UnsafeMeta` containing `TransactionMetaV4` (V=4) and populate `PostTxApplyFeeChanges` with a non-empty `LedgerEntryChanges` slice (e.g., a fee account balance state change representing a refund). Call `TransformLedgerTransaction()`.
- **Steps**: (1) Create the test transaction with V4 meta and non-empty `PostTxApplyFeeChanges`. (2) Call `TransformLedgerTransaction()`. (3) Decode the returned `TxMeta` (base64 → `TransactionMeta`). (4) Decode the returned `TxFeeMeta` (base64 → `LedgerEntryChanges`). (5) Check all output fields for the presence of the post-apply fee changes.
- **Assertion**: Assert that the post-apply fee changes from `PostTxApplyFeeChanges` do NOT appear in any of `TxMeta`, `TxFeeMeta`, or any other output field — confirming the data is silently dropped. The test demonstrates that the `LedgerTransactionOutput` cannot preserve fee refund XDR for P23+ transactions.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestPostTxApplyFeeChangesDroppedFromLedgerTransaction"
**Test Language**: Go

### Demonstration

The test constructs an `ingest.LedgerTransaction` with V4 meta (P23+) and a non-empty `PostTxApplyFeeChanges` slice containing a fee account balance state→updated pair representing a 10,705 stroop refund. After calling `TransformLedgerTransaction()`, the test decodes `TxMeta` (base64 → `TransactionMeta`) and checks `TxChangesAfter`, then compares `TxFeeMeta` against the serialized post-apply changes. Neither field contains the refund data, and `LedgerTransactionOutput` has no other field to carry it — confirming the data is silently dropped.

### Test Body

```go
func TestPostTxApplyFeeChangesDroppedFromLedgerTransaction(t *testing.T) {
	// 1. Build a P23+ transaction with non-empty PostTxApplyFeeChanges
	//    representing a fee refund (account balance state → updated).
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
						Balance:   4067134531458, // refund of 10,705 stroops
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
			},
		},
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount1,
					SeqNum:        1,
					Fee:           100,
					Operations: []xdr.Operation{
						{Body: xdr.OperationBody{Type: xdr.OperationTypeCreateAccount,
							CreateAccountOp: &xdr.CreateAccountOp{
								Destination:     testAccount5ID,
								StartingBalance: 1000000000,
							}}},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			Result: xdr.TransactionResult{
				FeeCharged: 100,
				Result: xdr.TransactionResultResult{
					Code: xdr.TransactionResultCodeTxSuccess,
					Results: &[]xdr.OperationResult{
						{
							Tr: &xdr.OperationResultTr{
								Type: xdr.OperationTypeCreateAccount,
								CreateAccountResult: &xdr.CreateAccountResult{
									Code: 0,
								},
							},
						},
					},
				},
			},
		},
		PostTxApplyFeeChanges: postApplyChanges,
	}

	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 50000000,
			ScpValue:  xdr.StellarValue{CloseTime: 1700000000},
		},
	}

	// 2. Run the production transform
	output, err := TransformLedgerTransaction(tx, lhe)
	if err != nil {
		t.Fatalf("TransformLedgerTransaction returned error: %v", err)
	}

	// 3. Serialize the original PostTxApplyFeeChanges to base64 for comparison
	expectedB64, err := xdr.MarshalBase64(postApplyChanges)
	if err != nil {
		t.Fatalf("Failed to marshal expected PostTxApplyFeeChanges: %v", err)
	}

	// 4. Check every output field — the post-apply fee changes should appear
	//    somewhere if the data is being preserved.
	found := false

	// Check TxMeta: decode and see if postApplyChanges are embedded
	if output.TxMeta != "" {
		metaBytes, _ := base64.StdEncoding.DecodeString(output.TxMeta)
		var meta xdr.TransactionMeta
		if err := meta.UnmarshalBinary(metaBytes); err == nil {
			if v4, ok := meta.GetV4(); ok {
				if len(v4.TxChangesAfter) > 0 {
					afterB64, _ := xdr.MarshalBase64(v4.TxChangesAfter)
					if afterB64 == expectedB64 {
						found = true
					}
				}
			}
		}
	}

	// Check TxFeeMeta
	if output.TxFeeMeta == expectedB64 {
		found = true
	}

	// LedgerTransactionOutput only has: TxEnvelope, TxResult, TxMeta, TxFeeMeta,
	// TxLedgerHistory, ClosedAt, LedgerSequence — none designed for post-apply changes.

	if found {
		t.Error("PostTxApplyFeeChanges unexpectedly found in output — hypothesis disproven")
	} else {
		t.Errorf("BUG CONFIRMED: PostTxApplyFeeChanges (P23+ fee refund data) is silently dropped. "+
			"PostTxApplyFeeChanges had %d entries (base64 length %d). "+
			"TxMeta does NOT contain post-apply fee changes. "+
			"TxFeeMeta does NOT contain post-apply fee changes. "+
			"LedgerTransactionOutput has no field for post-apply fee XDR.",
			len(postApplyChanges), len(expectedB64))
	}
}
```

### Test Output

```
=== RUN   TestPostTxApplyFeeChangesDroppedFromLedgerTransaction
    data_integrity_poc_test.go:204: BUG CONFIRMED: PostTxApplyFeeChanges (P23+ fee refund data) is silently dropped. PostTxApplyFeeChanges had 2 entries (base64 length 264). TxMeta does NOT contain post-apply fee changes. TxFeeMeta does NOT contain post-apply fee changes. LedgerTransactionOutput has no field for post-apply fee XDR.
--- FAIL: TestPostTxApplyFeeChangesDroppedFromLedgerTransaction (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.733s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-11
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Exercises claimed bug?** PARTIAL — I reproduced the factual omission with a corrected local PoC: `TransformLedgerTransaction()` serializes `xdr.MarshalBase64(transaction.UnsafeMeta)` into `tx_meta` and `xdr.MarshalBase64(transaction.FeeChanges)` into `tx_fee_meta`, so `PostTxApplyFeeChanges` is absent from the exported row.
2. **Realistic preconditions?** YES — protocol-23+ Soroban transactions can populate `LedgerTransaction.PostTxApplyFeeChanges`, and the upstream ingest reader surfaces that field separately from `UnsafeMeta`.
3. **Bug or by design?** BY DESIGN — the raw `ledger_transaction` schema only exposes `tx_meta` and `tx_fee_meta`, and both the current repository and released `stellar-etl` v2.8.11 keep the same contract. No existing field is misserialized: `tx_meta` remains raw `TransactionMeta`, and `tx_fee_meta` remains raw `FeeChanges`.
4. **Does the PoC demonstrate wrong output?** NO — it demonstrates that there is no separate column for a third low-level blob. It does not show any emitted field contains an incorrect value relative to its documented source.
5. **Impact match / in scope?** NO — as framed, this is a schema-expansion / missing-capability complaint, not a confirmed data-integrity defect in the exported fields.
6. **Is the test itself correct?** INSUFFICIENT — the original PoC hard-fails when `PostTxApplyFeeChanges` is not found anywhere, but it never establishes a documented invariant that `ledger_transaction` must round-trip all three ingest low-level buckets. A corrected test can prove omission, not corruption.
7. **Alternative explanation?** YES — protocol 23 changed where refund XDR lives, while this exporter intentionally continues to serialize only the pre-existing `UnsafeMeta` and `FeeChanges` blobs.
8. **Novelty?** NOT RELEVANT after rejection.

### Rejection Reason

`TransformLedgerTransaction()` faithfully exports the two raw blobs that `LedgerTransactionOutput` defines: `UnsafeMeta` as `tx_meta` and `FeeChanges` as `tx_fee_meta`. The PoC shows that no dedicated `PostTxApplyFeeChanges` column exists, but that is a schema gap rather than corruption of any existing exported field.

### Failed Checks

3, 4, 5, 6, 7
