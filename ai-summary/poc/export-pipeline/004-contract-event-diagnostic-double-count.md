# H004: Contract-event export can double-count events already present in diagnostics

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_contract_events` should emit each underlying contract event once, or it should preserve enough provenance to let downstream consumers distinguish "operation event" rows from "diagnostic mirror" rows without double-counting. A single emitted contract event should not become two history rows merely because the metadata also exposes it through the diagnostic stream.

## Mechanism

`TransformContractEvent()` blindly appends `TransactionEvents`, `OperationEvents`, and `DiagnosticEvents` into one output slice. Upstream `GetDiagnosticEvents()` explicitly warns that, for smart-contract transactions, diagnostic events **may include contract events as well** and callers should avoid double-counting them; the ETL does not deduplicate or tag the source stream, so mirrored events become duplicate exported rows.

## Trigger

Run `export_contract_events` on a smart-contract transaction whose transaction meta includes contract events inside `DiagnosticEvents` in addition to the normal contract-event stream. The exporter will append both copies and overstate the number of contract events for that transaction.

## Target Code

- `internal/transform/contract_events.go:TransformContractEvent:21-65` — concatenates transaction, operation, and diagnostic event arrays with no dedupe or source column
- `internal/transform/contract_events.go:parseDiagnosticEvent:167-241` — normalizes all three streams to the same `ContractEventOutput` shape
- `internal/transform/contract_events_test.go:58-145` — current fixture already encodes the duplicated-row pattern for v3/v4 metadata

## Evidence

`go doc -src github.com/stellar/go-stellar-sdk/xdr.TransactionMeta.GetDiagnosticEvents` and the ingest wrapper both document that `DiagnosticEvents` may include contract events and that callers "should be careful not to double count diagnostic events and contract events in that case". The ETL currently does the exact opposite by unioning both streams into the same table.

## Anti-Evidence

If a node or metadata configuration emits strictly non-contract diagnostic events, no duplication occurs. The bug depends on the documented configuration where diagnostics mirror contract events, but that configuration is explicitly supported upstream rather than theoretical.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `GetTransactionEvents()` in the upstream SDK through `TransformContractEvent()`. For V3 metadata (Soroban transactions), the SDK's `GetTransactionEvents()` populates `OperationEvents[0]` from `SorobanMeta.Events` and `DiagnosticEvents` from `SorobanMeta.DiagnosticEvents`. The SDK explicitly documents that diagnostic events "MAY include contract events as well" and warns callers to "be careful not to double count." The ETL ignores this warning and concatenates all arrays into a single output slice. The existing test fixture confirms this: it constructs identical events in both arrays and expects 2 output rows for the same underlying event, with differing `OperationID` values (one set, one null).

### Code Paths Examined

- `go-stellar-sdk/ingest/ledger_transaction.go:GetTransactionEvents:278-320` — For V3: populates `OperationEvents[0]` from `GetContractEvents()` (→ `SorobanMeta.Events`) and `DiagnosticEvents` from `GetDiagnosticEvents()` (→ `SorobanMeta.DiagnosticEvents`). For V4: populates all three arrays from raw XDR. No deduplication in either case.
- `go-stellar-sdk/xdr/transaction_meta.go:GetDiagnosticEvents:35-49` — Explicit SDK warning at lines 33-36: "the list of generated diagnostic events MAY include contract events...should be careful not to double count"
- `internal/transform/contract_events.go:TransformContractEvent:21-67` — Three sequential loops concatenate `TransactionEvents` (lines 31-39), `OperationEvents` (lines 42-56), and `DiagnosticEvents` (lines 58-65) into one slice with no dedup
- `internal/transform/contract_events_test.go:150-208` — V3 test fixture places the same ContractEvent (type=Diagnostic, ContractId=zero, topics=[true], data=true) in both `SorobanMeta.Events` and `SorobanMeta.DiagnosticEvents`
- `internal/transform/contract_events_test.go:58-92` — Expected V3 output: 2 rows for 1 underlying event — first row has `OperationID` set (from OperationEvents loop), second has null `OperationID` (from DiagnosticEvents loop)

### Findings

1. **V3 double-counting is confirmed and tested**: The test fixture creates identical event data in `SorobanMeta.Events` and `SorobanMeta.DiagnosticEvents`. `GetTransactionEvents()` returns both arrays unmodified. `TransformContractEvent()` processes both, producing 2 rows with differing `OperationID` for the same underlying event. This is exactly the pattern the SDK warns against.

2. **V4 overlap is plausible**: For V4 Soroban transactions, `DiagnosticEvents` may also mirror `Operations[i].Events` depending on node configuration. The ETL would produce duplicates in this case too. The V4 test fixture uses distinct event types across the three arrays, so V4 double-counting is not exercised by tests but is architecturally possible.

3. **No dedup key or source column**: `ContractEventOutput` has no field indicating which stream (TransactionEvents/OperationEvents/DiagnosticEvents) an event originated from. Downstream consumers in BigQuery cannot distinguish or deduplicate.

4. **Inconsistent OperationID on duplicates**: The same underlying event gets `OperationID` set when processed through the OperationEvents loop but null when processed through the DiagnosticEvents loop. This means the duplicate rows are not even identical — they differ in a semantically important field.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go`
- **Setup**: Construct a V3 `ingest.LedgerTransaction` where `SorobanMeta.Events` contains a contract event (type=Contract) and `SorobanMeta.DiagnosticEvents` contains a DiagnosticEvent wrapping an identical ContractEvent. This mirrors real-world behavior when diagnostic events include contract events.
- **Steps**: Call `TransformContractEvent(transaction, header)` and count the output rows.
- **Assertion**: Assert that the number of output rows equals the number of *unique* events, not the sum of all arrays. Alternatively, assert that each output row has a `source_stream` field or that duplicate events are tagged/filtered. The existing test at lines 58-92 already demonstrates the bug — it expects 2 rows for 1 unique event.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractEventDiagnosticDoubleCount"
**Test Language**: Go

### Demonstration

The test constructs a V3 Soroban transaction with a single contract event placed in both `SorobanMeta.Events` and `SorobanMeta.DiagnosticEvents`, mirroring the documented upstream behavior. `TransformContractEvent()` produces 2 output rows for this 1 unique event — one with `OperationID` set (from the OperationEvents loop) and one with null `OperationID` (from the DiagnosticEvents loop). This confirms the double-counting bug and the inconsistent OperationID on duplicate rows.

### Test Body

```go
func TestContractEventDiagnosticDoubleCount(t *testing.T) {
	hardCodedBool := true

	// Construct a single contract event that will appear in BOTH
	// SorobanMeta.Events and SorobanMeta.DiagnosticEvents.
	// This mirrors real-world behavior where diagnostic events include contract events.
	sharedEvent := xdr.ContractEvent{
		Ext:        xdr.ExtensionPoint{V: 0},
		ContractId: &xdr.ContractId{},
		Type:       xdr.ContractEventTypeContract,
		Body: xdr.ContractEventBody{
			V: 0,
			V0: &xdr.ContractEventV0{
				Topics: []xdr.ScVal{
					{
						Type: xdr.ScValTypeScvBool,
						B:    &hardCodedBool,
					},
				},
				Data: xdr.ScVal{
					Type: xdr.ScValTypeScvBool,
					B:    &hardCodedBool,
				},
			},
		},
	}

	// Place the SAME event in both SorobanMeta.Events and SorobanMeta.DiagnosticEvents.
	// The SDK's GetTransactionEvents() will return it in OperationEvents (from Events)
	// AND in DiagnosticEvents (from DiagnosticEvents).
	txMetaV3 := xdr.TransactionMetaV3{
		SorobanMeta: &xdr.SorobanTransactionMeta{
			Events: []xdr.ContractEvent{sharedEvent},
			DiagnosticEvents: []xdr.DiagnosticEvent{
				{
					InSuccessfulContractCall: true,
					Event:                    sharedEvent,
				},
			},
		},
	}

	hardCodedMemoText := "test"
	hardCodedTransactionHash := xdr.Hash([32]byte{0xaa})
	genericResultResults := &[]xdr.OperationResult{
		{
			Tr: &xdr.OperationResultTr{
				Type: xdr.OperationTypeCreateAccount,
				CreateAccountResult: &xdr.CreateAccountResult{
					Code: 0,
				},
			},
		},
	}

	destination := xdr.MuxedAccount{
		Type:    xdr.CryptoKeyTypeKeyTypeEd25519,
		Ed25519: &xdr.Uint256{1, 2, 3},
	}

	transaction := ingest.LedgerTransaction{
		Index: 1,
		UnsafeMeta: xdr.TransactionMeta{
			V:  3,
			V3: &txMetaV3,
		},
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					Ext: xdr.TransactionExt{
						V: 1,
						SorobanData: &xdr.SorobanTransactionData{
							Ext:         xdr.SorobanTransactionDataExt{},
							Resources:   xdr.SorobanResources{},
							ResourceFee: 0,
						},
					},
					SourceAccount: testAccount1,
					SeqNum:        112351890582290871,
					Memo: xdr.Memo{
						Type: xdr.MemoTypeMemoText,
						Text: &hardCodedMemoText,
					},
					Fee: 90000,
					Cond: xdr.Preconditions{
						Type: xdr.PreconditionTypePrecondTime,
						TimeBounds: &xdr.TimeBounds{
							MinTime: 0,
							MaxTime: 1594272628,
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
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: hardCodedTransactionHash,
			Result: xdr.TransactionResult{
				FeeCharged: 300,
				Result: xdr.TransactionResultResult{
					Code:    xdr.TransactionResultCodeTxFailed,
					Results: genericResultResults,
				},
			},
		},
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 30521816,
			ScpValue:  xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	// Run the production transform
	output, err := TransformContractEvent(transaction, header)
	if err != nil {
		t.Fatalf("TransformContractEvent returned unexpected error: %v", err)
	}

	// There is only 1 unique underlying contract event, but the transform
	// processes it from both OperationEvents and DiagnosticEvents arrays.
	uniqueEventCount := 1

	// BUG: The transform produces 2 rows for 1 unique event.
	// The OperationEvents loop emits the event with OperationID set,
	// and the DiagnosticEvents loop emits it again with OperationID null.
	if len(output) <= uniqueEventCount {
		t.Fatalf("Expected double-counted output (>%d rows), but got %d rows. "+
			"If this fails, the deduplication bug may have been fixed.", uniqueEventCount, len(output))
	}

	t.Logf("BUG CONFIRMED: 1 unique event produced %d output rows (expected 1)", len(output))

	// Further verify the duplicate rows differ in OperationID — showing inconsistent data
	hasOperationID := false
	hasNullOperationID := false
	for _, row := range output {
		if row.OperationID.Valid {
			hasOperationID = true
		} else {
			hasNullOperationID = true
		}
	}

	if !hasOperationID || !hasNullOperationID {
		t.Errorf("Expected duplicate rows to have inconsistent OperationID "+
			"(one set, one null), but got hasOperationID=%v, hasNullOperationID=%v",
			hasOperationID, hasNullOperationID)
	}

	t.Logf("Duplicate rows have inconsistent OperationID: one has it set, one has it null")
	t.Logf("Row 0 OperationID: valid=%v value=%v", output[0].OperationID.Valid, output[0].OperationID.Int64)
	t.Logf("Row 1 OperationID: valid=%v value=%v", output[1].OperationID.Valid, output[1].OperationID.Int64)
}
```

### Test Output

```
=== RUN   TestContractEventDiagnosticDoubleCount
    data_integrity_poc_test.go:159: BUG CONFIRMED: 1 unique event produced 2 output rows (expected 1)
    data_integrity_poc_test.go:178: Duplicate rows have inconsistent OperationID: one has it set, one has it null
    data_integrity_poc_test.go:179: Row 0 OperationID: valid=true value=131090201534533633
    data_integrity_poc_test.go:180: Row 1 OperationID: valid=false value=0
--- PASS: TestContractEventDiagnosticDoubleCount (0.00s)
PASS
```
