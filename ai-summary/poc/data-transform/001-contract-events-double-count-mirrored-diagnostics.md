# H001: `TransformContractEvent()` double-counts contract events mirrored into diagnostics

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_contract_events` should contain one exported row per emitted event. If txmeta exposes the same contract event through both the canonical contract-event stream and the optional diagnostic-event stream, the transform should preserve only one copy or otherwise distinguish the two sources so downstream consumers do not count the same event twice.

## Mechanism

`TransformContractEvent()` appends every event from `TransactionEvents`, every event from `OperationEvents`, and every entry from `DiagnosticEvents` into one flat slice without any deduplication or source guard. The upstream SDK explicitly documents that diagnostic events may include contract events as well, so a txmeta producer that mirrors contract events into diagnostics will cause the ETL to emit duplicate rows with the same payload, contract ID, topics, data, and success flags.

## Trigger

Export a Soroban transaction whose txmeta includes a contract event in `OperationEvents` and also repeats that same event in `DiagnosticEvents` (the SDK documents this as configuration-dependent). The current transform will append both and produce two `history_contract_events` rows for one on-chain event.

## Target Code

- `internal/transform/contract_events.go:21-67` — `TransformContractEvent()` appends transaction, operation, and diagnostic streams into one slice with no dedupe.
- `internal/transform/contract_events.go:167-241` — `parseDiagnosticEvent()` normalizes all three sources into the same output shape, so mirrored events become indistinguishable duplicates.

## Evidence

The code path is straightforward: every `transactionEvents.TransactionEvents`, every `transactionEvents.OperationEvents[i]`, and every `transactionEvents.DiagnosticEvents` entry is appended. The upstream `ingest.LedgerTransaction.GetDiagnosticEvents()` documentation warns that diagnostic events "MAY include contract events as well" and that consumers should be careful not to double count them, but `TransformContractEvent()` does not apply any such guard.

## Anti-Evidence

If txmeta is generated without duplicating contract events into diagnostics, no duplicate rows appear. Some downstream users may also notice duplicates by joining on `contract_event_xdr`, but the exported table itself currently provides no source discriminator or dedupe.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `GetTransactionEvents()` in the SDK (`go-stellar-sdk/ingest/ledger_transaction.go:278-312`) through to `TransformContractEvent()` (`internal/transform/contract_events.go:21-67`). For V3 txmeta, the SDK populates `OperationEvents[0]` from `SorobanMeta.Events` and `DiagnosticEvents` from `SorobanMeta.DiagnosticEvents`. The SDK documentation at line 260-263 explicitly warns that DiagnosticEvents "MAY include contract events as well" and that consumers should "be careful not to double count." `TransformContractEvent()` ignores this warning — it runs three independent loops over TransactionEvents, OperationEvents, and DiagnosticEvents, appending all results to one flat slice with no deduplication. The existing unit test (contract_events_test.go:150-459) constructs V3 input that deliberately places the same event in both `Events` and `DiagnosticEvents`, and the expected output at lines 58-92 asserts 2 rows for this single on-chain event, confirming the duplication is baked into the test suite.

### Code Paths Examined

- `go-stellar-sdk/ingest/ledger_transaction.go:278-312` — `GetTransactionEvents()` returns raw XDR arrays without deduplication; V3 fills OperationEvents from SorobanMeta.Events and DiagnosticEvents from SorobanMeta.DiagnosticEvents
- `go-stellar-sdk/ingest/ledger_transaction.go:260-263` — `GetDiagnosticEvents()` documentation: "diagnostic events MAY include contract events as well; Users... should be careful not to double count"
- `go-stellar-sdk/xdr/transaction_meta.go:35-50` — `GetDiagnosticEvents()` returns raw DiagnosticEvents from both V3 and V4 txmeta
- `internal/transform/contract_events.go:21-67` — `TransformContractEvent()` processes all three arrays in sequence with no overlap check
- `internal/transform/contract_events.go:42-56` — OperationEvents loop wraps each ContractEvent as DiagnosticEvent and appends
- `internal/transform/contract_events.go:58-65` — DiagnosticEvents loop appends directly, potentially re-processing the same events
- `internal/transform/contract_events_test.go:150-208` — V3 test input places the same event in both SorobanMeta.Events and SorobanMeta.DiagnosticEvents
- `internal/transform/contract_events_test.go:58-92` — Expected V3 output asserts 2 rows for 1 on-chain event, confirming the duplication

### Findings

1. **V3 duplication (confirmed by test)**: When stellar-core has `ENABLE_SOROBAN_DIAGNOSTIC_EVENTS=true` (standard for Horizon/indexer nodes), `SorobanMeta.DiagnosticEvents` is a superset of `SorobanMeta.Events`. Every contract event appears in both. `TransformContractEvent()` processes both → each contract event produces 2 output rows. The test at line 58-92 explicitly asserts this: the two rows are identical except that the OperationEvents-sourced row has an OperationID and the DiagnosticEvents-sourced row does not.

2. **V4 duplication (same risk)**: For V4 txmeta (protocol 23+/CAP-67), the same pattern applies. `txMeta.Operations[i].Events` and `txMeta.DiagnosticEvents` can overlap. The SDK warning applies to both V3 and V4. The V4 test case uses different event types across the arrays so it doesn't exercise the overlap, but the code has no guard for V4 either.

3. **No deduplication or source tagging**: The output rows carry no field indicating whether they came from OperationEvents or DiagnosticEvents. The only observable difference is that OperationEvents-sourced rows have OperationID set while DiagnosticEvents-sourced rows do not — but OperationID absence could also mean a transaction-level event, making it unreliable as a discriminator.

4. **Impact on downstream analytics**: `history_contract_events` rows would be double-counted by any COUNT, SUM, or GROUP BY query. Event-driven pipelines would process the same event twice. There is no reliable way for downstream consumers to identify and exclude the duplicates.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go`
- **Setup**: The existing V3 test fixture already demonstrates the issue. Create a focused test that (a) constructs a V3 Soroban transaction with 1 contract event in `SorobanMeta.Events` and the same event mirrored in `SorobanMeta.DiagnosticEvents`, (b) calls `TransformContractEvent()`, and (c) asserts the output length is 2 when it should be 1 (or verifies the duplication by comparing ContractEventXDR fields).
- **Steps**: (1) Build SorobanTransactionMeta with Events=[E1] and DiagnosticEvents=[DiagWrap(E1)]. (2) Call TransformContractEvent. (3) Count output rows. (4) Compare ContractEventXDR values of the two rows — they will be identical (the existing test fixture at lines 73 and 90 shows this: both rows have the same ContractEventXDR base64 string).
- **Assertion**: Assert that `len(output) == 2` (demonstrating the bug) when `len(SorobanMeta.Events) == 1` and `len(SorobanMeta.DiagnosticEvents) == 1` contain the same event. The correct behavior would be `len(output) == 1`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractEventsDoubleCountMirroredDiagnostics"
**Test Language**: Go

### Demonstration

The test constructs a V3 Soroban transaction with exactly 1 contract event placed in both `SorobanMeta.Events` and `SorobanMeta.DiagnosticEvents` (mirroring what stellar-core produces with `ENABLE_SOROBAN_DIAGNOSTIC_EVENTS=true`). Calling `TransformContractEvent()` produces 2 output rows instead of 1. Both rows carry identical `ContractEventXDR` (`AAAAAQAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAEAAAAAAAAAAQAAAAAAAAAB`), making them indistinguishable to downstream consumers. The only difference is the OperationEvents-sourced row has OperationID set while the DiagnosticEvents-sourced row does not — an unreliable discriminator since transaction-level events also lack OperationID.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestContractEventsDoubleCountMirroredDiagnostics demonstrates that
// TransformContractEvent() produces 2 output rows for a single on-chain
// contract event when the same event appears in both SorobanMeta.Events
// and SorobanMeta.DiagnosticEvents (the standard layout when stellar-core
// has ENABLE_SOROBAN_DIAGNOSTIC_EVENTS=true).
//
// The SDK documentation explicitly warns: "diagnostic events MAY include
// contract events as well; users should be careful not to double count."
// TransformContractEvent ignores this warning.
func TestContractEventsDoubleCountMirroredDiagnostics(t *testing.T) {
	// 1. Construct a single contract event
	hardCodedBool := true
	singleEvent := xdr.ContractEvent{
		Ext:        xdr.ExtensionPoint{V: 0},
		ContractId: &xdr.ContractId{},
		Type:       xdr.ContractEventTypeDiagnostic,
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

	// 2. Place the SAME event in both SorobanMeta.Events (→ OperationEvents)
	//    and SorobanMeta.DiagnosticEvents. This mirrors what stellar-core
	//    produces when diagnostic events are enabled.
	txMeta := xdr.TransactionMeta{
		V: 3,
		V3: &xdr.TransactionMetaV3{
			SorobanMeta: &xdr.SorobanTransactionMeta{
				Events:           []xdr.ContractEvent{singleEvent},
				DiagnosticEvents: []xdr.DiagnosticEvent{{InSuccessfulContractCall: true, Event: singleEvent}},
			},
		},
	}

	hardCodedMemoText := "test"
	destination := xdr.MuxedAccount{
		Type:    xdr.CryptoKeyTypeKeyTypeEd25519,
		Ed25519: &xdr.Uint256{1, 2, 3},
	}

	transaction := ingest.LedgerTransaction{
		Index:      1,
		UnsafeMeta: txMeta,
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
					SeqNum:        1,
					Memo: xdr.Memo{
						Type: xdr.MemoTypeMemoText,
						Text: &hardCodedMemoText,
					},
					Fee: 100,
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
		Result: utils.CreateSampleResultMeta(false, 1).Result,
	}

	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 100,
			ScpValue:  xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	// 3. Call the production code path
	output, err := TransformContractEvent(transaction, lhe)
	if err != nil {
		t.Fatalf("TransformContractEvent returned unexpected error: %v", err)
	}

	// The input has exactly 1 unique on-chain event. Correct behavior would
	// produce 1 output row. The bug causes 2 rows to be emitted.
	uniqueEventCount := 1
	if len(output) == uniqueEventCount {
		t.Fatal("Expected duplication bug but output has correct count — hypothesis disproved")
	}

	// 4. Assert the duplication: 2 rows for 1 event
	if len(output) != 2 {
		t.Fatalf("Expected 2 output rows (demonstrating duplication), got %d", len(output))
	}
	t.Logf("BUG CONFIRMED: %d unique on-chain event produced %d output rows", uniqueEventCount, len(output))

	// 5. Verify the two rows carry identical event payloads (same ContractEventXDR)
	if output[0].ContractEventXDR == output[1].ContractEventXDR {
		t.Logf("Both rows have identical ContractEventXDR: %s", output[0].ContractEventXDR)
		t.Log("Downstream consumers cannot distinguish duplicates — double-counting is unavoidable")
	} else {
		t.Logf("Row 0 XDR: %s", output[0].ContractEventXDR)
		t.Logf("Row 1 XDR: %s", output[1].ContractEventXDR)
		t.Log("XDR differs due to DiagnosticEvent wrapper, but inner event payload is identical")
	}

	// Verify the OperationID pattern that distinguishes the sources
	if output[0].OperationID.Valid && !output[1].OperationID.Valid {
		t.Log("Row 0 (from OperationEvents): has OperationID")
		t.Log("Row 1 (from DiagnosticEvents): no OperationID")
	} else if !output[0].OperationID.Valid && output[1].OperationID.Valid {
		t.Log("Row 0 (from DiagnosticEvents): no OperationID")
		t.Log("Row 1 (from OperationEvents): has OperationID")
	}

	// Verify all other fields match (same event data)
	if output[0].ContractId != output[1].ContractId {
		t.Errorf("ContractId mismatch: %q vs %q", output[0].ContractId, output[1].ContractId)
	}
	if output[0].TypeString != output[1].TypeString {
		t.Errorf("TypeString mismatch: %q vs %q", output[0].TypeString, output[1].TypeString)
	}
	if output[0].TransactionHash != output[1].TransactionHash {
		t.Errorf("TransactionHash mismatch: %q vs %q", output[0].TransactionHash, output[1].TransactionHash)
	}
}
```

### Test Output

```
=== RUN   TestContractEventsDoubleCountMirroredDiagnostics
    data_integrity_poc_test.go:133: BUG CONFIRMED: 1 unique on-chain event produced 2 output rows
    data_integrity_poc_test.go:139: Both rows have identical ContractEventXDR: AAAAAQAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAEAAAAAAAAAAQAAAAAAAAAB
    data_integrity_poc_test.go:140: Downstream consumers cannot distinguish duplicates — double-counting is unavoidable
    data_integrity_poc_test.go:150: Row 0 (from OperationEvents): has OperationID
    data_integrity_poc_test.go:151: Row 1 (from DiagnosticEvents): no OperationID
--- PASS: TestContractEventsDoubleCountMirroredDiagnostics (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.706s
```
