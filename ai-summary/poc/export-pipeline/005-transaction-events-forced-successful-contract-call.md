# H005: Transaction-level fee events are mislabeled as successful contract calls

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_contract_events` should preserve the true `in_successful_contract_call` semantics of exported rows. Transaction-level fee/refund events, especially on failed Soroban transactions, should not be labeled as if they happened inside a successful contract call.

## Mechanism

Every `TransactionEvent` is rewritten into a synthetic `DiagnosticEvent` with `InSuccessfulContractCall: true`, regardless of transaction outcome or event stage. `parseDiagnosticEvent()` then copies that hard-coded boolean straight into `ContractEventOutput`, so failed-transaction fee events export with a contradictory combination like `successful=false` and `in_successful_contract_call=true`.

## Trigger

Run `export_contract_events` on a failed Soroban transaction that still emits transaction-level fee/refund events in `TransactionMetaV4.Events`. The exported row will claim `in_successful_contract_call=true` even though the enclosing transaction failed and the event did not come from a successful contract invocation.

## Target Code

- `internal/transform/contract_events.go:transactionEvent2DiagnosticEvent:139-149` — hard-codes `InSuccessfulContractCall: true` for every `TransactionEvent`
- `internal/transform/contract_events.go:TransformContractEvent:31-38` — applies that conversion to all transaction-level events
- `internal/transform/contract_events.go:parseDiagnosticEvent:194-239` — persists the fabricated boolean to output
- `internal/transform/contract_events_test.go:94-109` — repository fixture already expects a failed transaction row with `InSuccessfulContractCall: true`

## Evidence

The current helper comment claims transaction events are only emitted in successful contract calls, but upstream XDR describes transaction events as transaction-level events such as fee payment/refund. The repository's own test data demonstrates the contradiction: it exports a row with `Successful: false` while keeping `InSuccessfulContractCall: true`.

## Anti-Evidence

Operation-scoped `ContractEvent` rows in `OperationEvents` are expected to come from successful contract execution, so the hard-coded `true` is not obviously wrong for that stream. The corruption is specific to transaction-level events that are not themselves contract-call events.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full data path from `TransformContractEvent()` through `transactionEvent2DiagnosticEvent()` and `parseDiagnosticEvent()`. The XDR definition of `TransactionMetaV4.Events` explicitly states these are "Used for transaction-level events (like fee payment)" — not contract-call events. The `TransactionEventStage` enum has stages like `BEFORE_ALL_TXS` ("before any one of the transactions has its operations applied"), which by definition cannot occur inside a contract call. The hard-coded `InSuccessfulContractCall: true` in `transactionEvent2DiagnosticEvent()` is based on a factually incorrect comment and produces semantically wrong output for all transaction-level events.

### Code Paths Examined

- `internal/transform/contract_events.go:transactionEvent2DiagnosticEvent:139-149` — Hard-codes `InSuccessfulContractCall: true` with comment "TransactionEvents are only emitted in successful contract calls" which contradicts the XDR specification
- `internal/transform/contract_events.go:TransformContractEvent:31-38` — Iterates `transactionEvents.TransactionEvents` and passes each through the lossy conversion
- `internal/transform/contract_events.go:parseDiagnosticEvent:167-242` — Copies `diagnosticEvent.InSuccessfulContractCall` (the fabricated `true`) into `ContractEventOutput.InSuccessfulContractCall` at line 194/230
- `internal/transform/contract_events.go:contractEvent2DiagnosticEvent:152-159` — Same pattern for operation-level `ContractEvent`, but this one is defensible since operation events ARE from successful execution
- `internal/transform/contract_events_test.go:93-110` — Test expects `Successful: false` + `InSuccessfulContractCall: true` for a `TransactionEvent` with `Stage: BEFORE_ALL_TXS` on a `TxFailed` transaction — confirming the contradiction in the test fixture
- `internal/transform/schema.go:ContractEventOutput:641-657` — Schema exposes `in_successful_contract_call` as a `bool` JSON field consumed by BigQuery
- XDR `TransactionMetaV4` struct comment: `TransactionEvent events<>; // Used for transaction-level events (like fee payment)` — confirms these are NOT contract-call events
- XDR `TransactionEventStage` enum: three stages (`BEFORE_ALL_TXS=0`, `AFTER_TX=1`, `AFTER_ALL_TXS=2`) describe transaction/ledger lifecycle phases, not contract invocation context

### Findings

1. **Factually incorrect code comment**: The comment at line 143 states "TransactionEvents are only emitted in successful contract calls." The XDR definition of `TransactionMetaV4.Events` says the opposite: these are "Used for transaction-level events (like fee payment)." The `TransactionEventStage` enum describes ledger/transaction lifecycle phases (before all txs, after tx, after all txs), none of which are inside a contract call.

2. **Hard-coded wrong value**: `transactionEvent2DiagnosticEvent()` unconditionally sets `InSuccessfulContractCall: true` for every `TransactionEvent`. Since `TransactionEvent` represents transaction-level lifecycle events (fee payment, fee refund), not contract invocations, this value is semantically wrong for all events in this category.

3. **Test confirms the contradiction**: The test fixture at lines 94-109 expects `Successful: false` combined with `InSuccessfulContractCall: true` for a `TransactionEvent` at stage `BEFORE_ALL_TXS` on a `TxFailed` transaction. An event that happened "before any transaction operations were applied" on a failed transaction cannot have occurred "in a successful contract call."

4. **Distinct from the Stage-dropping issue (H003/fail 011)**: The previously investigated hypothesis about dropping the `Stage` field was rejected as working-as-designed. This hypothesis addresses a different problem: the fabricated `InSuccessfulContractCall` boolean value. Dropping `Stage` was an explicit design choice with a documenting comment. Setting `InSuccessfulContractCall: true` is based on a factually wrong assumption in the comment.

5. **Downstream impact**: BigQuery consumers querying `WHERE in_successful_contract_call = true` will include transaction-level fee events that never occurred in a contract call. Queries filtering on `WHERE successful = false AND in_successful_contract_call = true` would return contradictory rows.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go`
- **Setup**: Create a `TransactionMetaV4` with a `TransactionEvent` at stage `BEFORE_ALL_TXS` on a failed transaction (`TransactionResultCodeTxFailed`). The existing test helper `makeContractEventTestInput()` already does this for the second test case (index 1, lines 210-308 and 389-442).
- **Steps**: Call `TransformContractEvent()` with the test transaction and inspect the output for events sourced from `transactionEvents.TransactionEvents`.
- **Assertion**: Assert that for the `TransactionEvent`-sourced output row, `InSuccessfulContractCall` is `false` (not `true`). Currently, the test at lines 94-109 asserts the opposite (`true`), confirming the bug. A corrected implementation should set `InSuccessfulContractCall: false` for all `TransactionEvent` entries, since they represent transaction-level lifecycle events, not contract-call events.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTransactionEventForcedSuccessfulContractCall"
**Test Language**: Go

### Demonstration

The test constructs a failed Soroban transaction (`TxFailed`) carrying a `TransactionEvent` at stage `BEFORE_ALL_TXS` — a transaction-level lifecycle phase that by definition occurs before any operations are applied and cannot be inside a contract call. After running `TransformContractEvent()`, the output row has `Successful=false` (correct) but `InSuccessfulContractCall=true` (wrong). This confirms that `transactionEvent2DiagnosticEvent()` unconditionally fabricates `InSuccessfulContractCall: true` for all `TransactionEvent` entries, producing semantically contradictory output for downstream BigQuery consumers.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestTransactionEventForcedSuccessfulContractCall demonstrates that
// transactionEvent2DiagnosticEvent hard-codes InSuccessfulContractCall=true
// for every TransactionEvent, even when the transaction failed and the event
// stage is BEFORE_ALL_TXS — a phase that by definition cannot occur inside
// a contract call.
func TestTransactionEventForcedSuccessfulContractCall(t *testing.T) {
	hardCodedBool := true

	// Build a TransactionEvent at stage BEFORE_ALL_TXS (fee payment phase)
	txEvent := xdr.TransactionEvent{
		Stage: xdr.TransactionEventStageTransactionEventStageBeforeAllTxs,
		Event: xdr.ContractEvent{
			Ext:        xdr.ExtensionPoint{V: 0},
			ContractId: &xdr.ContractId{},
			Type:       xdr.ContractEventTypeContract,
			Body: xdr.ContractEventBody{
				V: 0,
				V0: &xdr.ContractEventV0{
					Topics: []xdr.ScVal{{
						Type: xdr.ScValTypeScvBool,
						B:    &hardCodedBool,
					}},
					Data: xdr.ScVal{
						Type: xdr.ScValTypeScvBool,
						B:    &hardCodedBool,
					},
				},
			},
		},
	}

	// Build a FAILED transaction carrying this TransactionEvent via MetaV4
	txMeta := xdr.TransactionMetaV4{
		Ext:             xdr.ExtensionPoint{},
		TxChangesBefore: xdr.LedgerEntryChanges{},
		Operations:      []xdr.OperationMetaV2{},
		TxChangesAfter:  xdr.LedgerEntryChanges{},
		SorobanMeta: &xdr.SorobanTransactionMetaV2{
			Ext: xdr.SorobanTransactionMetaExt{
				V: 1,
				V1: &xdr.SorobanTransactionMetaExtV1{},
			},
			ReturnValue: &xdr.ScVal{},
		},
		Events:           []xdr.TransactionEvent{txEvent},
		DiagnosticEvents: []xdr.DiagnosticEvent{},
	}

	genericResults := &[]xdr.OperationResult{{
		Tr: &xdr.OperationResultTr{
			Type: xdr.OperationTypeCreateAccount,
			CreateAccountResult: &xdr.CreateAccountResult{
				Code: 0,
			},
		},
	}}

	hardCodedHash := xdr.Hash([32]byte{0xab})
	destination := xdr.MuxedAccount{
		Type:    xdr.CryptoKeyTypeKeyTypeEd25519,
		Ed25519: &xdr.Uint256{1, 2, 3},
	}
	memoText := "poc-test"

	ledgerTx := ingest.LedgerTransaction{
		Index: 1,
		UnsafeMeta: xdr.TransactionMeta{
			V:  4,
			V4: &txMeta,
		},
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					Ext: xdr.TransactionExt{
						V: 1,
						SorobanData: &xdr.SorobanTransactionData{},
					},
					SourceAccount: testAccount1,
					SeqNum:        1,
					Memo:          xdr.Memo{Type: xdr.MemoTypeMemoText, Text: &memoText},
					Fee:           1000,
					Cond: xdr.Preconditions{
						Type:       xdr.PreconditionTypePrecondTime,
						TimeBounds: &xdr.TimeBounds{MinTime: 0, MaxTime: 1594272628},
					},
					Operations: []xdr.Operation{{
						SourceAccount: &testAccount2,
						Body: xdr.OperationBody{
							Type: xdr.OperationTypePathPaymentStrictReceive,
							PathPaymentStrictReceiveOp: &xdr.PathPaymentStrictReceiveOp{
								Destination: destination,
							},
						},
					}},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: hardCodedHash,
			Result: xdr.TransactionResult{
				FeeCharged: 300,
				Result: xdr.TransactionResultResult{
					Code:    xdr.TransactionResultCodeTxFailed, // Transaction FAILED
					Results: genericResults,
				},
			},
		},
	}

	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 30521816,
			ScpValue:  xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	// Run the production transform
	outputs, err := TransformContractEvent(ledgerTx, lhe)
	if err != nil {
		t.Fatalf("TransformContractEvent returned error: %v", err)
	}

	if len(outputs) == 0 {
		t.Fatal("expected at least one contract event output from TransactionEvents")
	}

	// The first output comes from the TransactionEvent (BEFORE_ALL_TXS stage).
	// The transaction FAILED, so this event cannot have occurred inside a
	// successful contract call.
	evt := outputs[0]

	if evt.Successful {
		t.Fatal("expected Successful=false for a TxFailed transaction")
	}

	// BUG: InSuccessfulContractCall is true even though the event is a
	// transaction-level event at BEFORE_ALL_TXS stage on a failed transaction.
	// A correct implementation would set InSuccessfulContractCall=false for
	// TransactionEvent entries.
	if evt.InSuccessfulContractCall {
		t.Errorf("DATA INTEGRITY BUG CONFIRMED: TransactionEvent at stage BEFORE_ALL_TXS "+
			"on a FAILED transaction has InSuccessfulContractCall=true. "+
			"Got Successful=%v, InSuccessfulContractCall=%v. "+
			"Transaction-level events (fee payment/refund) are NOT contract-call events.",
			evt.Successful, evt.InSuccessfulContractCall)
	}
}
```

### Test Output

```
=== RUN   TestTransactionEventForcedSuccessfulContractCall
    data_integrity_poc_test.go:151: DATA INTEGRITY BUG CONFIRMED: TransactionEvent at stage BEFORE_ALL_TXS on a FAILED transaction has InSuccessfulContractCall=true. Got Successful=false, InSuccessfulContractCall=true. Transaction-level events (fee payment/refund) are NOT contract-call events.
--- FAIL: TestTransactionEventForcedSuccessfulContractCall (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.797s
```
