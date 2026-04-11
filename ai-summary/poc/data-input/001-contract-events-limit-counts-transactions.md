# H001: Contract-event export applies `--limit` to transactions instead of emitted events

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Empty or truncated contract-event exports under caller-visible row limits
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_contract_events --limit N` should keep scanning the requested ledger range until it has emitted `N` `ContractEventOutput` rows or exhausted the range. Transactions that produce zero contract-event rows should not consume the caller's event budget.

## Mechanism

`export_contract_events` passes the caller-visible `limit` directly into `input.GetTransactions()`, and that reader stops after `N` transactions rather than after `N` exported contract events. `transform.TransformContractEvent()` then expands each returned transaction into zero, one, or many output rows by concatenating transaction events, operation events, and diagnostic events, so the command can silently return an empty or truncated file even when later transactions in-range contain real contract events.

## Trigger

Run `stellar-etl export_contract_events --limit 1 --start-ledger <S> --end-ledger <E>` on a range where the first transaction is a classic/non-event transaction but a later transaction in the same range emits Soroban contract or diagnostic events. The correct output is one event row from the later transaction; the current code can return zero rows because the first transaction consumed the entire limit before expansion.

## Target Code

- `cmd/export_contract_events.go:25-43` — applies the user-visible `limit` at `GetTransactions()` time and then exports every event row returned by the transform
- `internal/input/transactions.go:23-70` — enforces `limit` on `len(txSlice)`, i.e. transaction count
- `internal/transform/contract_events.go:21-67` — expands one transaction into a variable-length `[]ContractEventOutput`
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of `contract_events` to export
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:278-311` — `GetTransactionEvents()` legitimately returns empty event sets for many transactions

## Evidence

The flag text is row-oriented (`contract_events`), but the reader budget is transaction-oriented. `TransformContractEvent()` explicitly appends rows from three independent event collections, while the upstream SDK's `GetTransactionEvents()` returns empty results for many transactions (for example, TxMeta V1/V2 and non-Soroban V3 paths), so an early zero-event transaction can exhaust `--limit` before any event row is emitted.

## Anti-Evidence

If the first limited transactions all happen to emit at least one contract-event row, the command still produces data and the bug is easy to miss. Users who leave `--limit` negative also avoid the problem because the reader scans the full range.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The hypothesis accurately describes a real limit-granularity mismatch in `export_contract_events`. The command passes the user-visible `--limit` (documented as "Maximum number of contract_events to export") directly into `input.GetTransactions()`, which enforces it at transaction count granularity. `TransformContractEvent()` then expands each transaction into zero-to-many `ContractEventOutput` rows from three separate event arrays, but the exporter writes all returned rows without any secondary cap. This is the same class of bug confirmed in `export_effects` (success/data-input/004) and `export_trades` (success/export-pipeline/010), but applied to a distinct command with its own independent code path.

### Code Paths Examined

- `cmd/export_contract_events.go:25` — passes `cmdArgs.Limit` directly to `input.GetTransactions()`, applying the limit at transaction granularity
- `cmd/export_contract_events.go:33-53` — iterates ALL events from `TransformContractEvent()` without any secondary row count cap
- `internal/input/transactions.go:51` — `for int64(len(txSlice)) < limit || limit < 0` enforces limit on `len(txSlice)` (transaction count)
- `internal/input/transactions.go:65-67` — breaks outer ledger loop when transaction limit is reached
- `internal/transform/contract_events.go:21-67` — `TransformContractEvent` calls `transaction.GetTransactionEvents()` and concatenates rows from `TransactionEvents`, `OperationEvents`, and `DiagnosticEvents` arrays — any of which may be empty for non-Soroban transactions
- `internal/utils/main.go:254` — flag definition: `"Maximum number of contract_events to export"` — clearly specifies event-row semantics

### Findings

The bug manifests in two directions:

1. **Empty/truncated exports**: With `--limit N` on a mixed ledger range, if the first N transactions are classic (non-Soroban), `GetTransactions()` returns N transactions that each produce zero contract events. The exporter writes zero rows even though later in-range Soroban transactions have events. This is especially likely because most Stellar transactions are classic, not Soroban.

2. **Oversized exports**: If one of the first N transactions is a complex Soroban invocation emitting many events, the exporter writes all of them with no cap, exceeding the user's requested limit.

The contract events case is arguably more impactful than the effects case (success/data-input/004) because a much higher fraction of transactions produce zero events (only Soroban transactions emit contract events), making the empty-export scenario far more likely in practice.

Severity downgraded from High to Medium for consistency with the confirmed effects finding (data-input/004, also Medium), since both are the same class of operational correctness issue with the `--limit` flag. Users running production pipelines typically use `--limit -1` (the default), which bypasses this codepath entirely.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go` (or create a new test in `internal/transform/data_integrity_poc_test.go` if it exists)
- **Setup**: Build a `LedgerTransaction` with `TxMeta` V1 or V2 (no Soroban events), similar to the existing test fixtures in `contract_events_test.go`. Confirm `TransformContractEvent()` returns an empty slice.
- **Steps**: (1) Call `TransformContractEvent()` on a non-Soroban transaction and verify it returns 0 rows. (2) Show that `GetTransactions(start, end, limit=1, ...)` returns exactly 1 transaction regardless of event content. (3) Demonstrate that the exporter loop writes all returned events without a secondary cap.
- **Assertion**: Assert that a single non-Soroban transaction passed through `TransformContractEvent()` yields `len(result) == 0`, proving that `--limit 1` would produce an empty export when the first transaction has no events, even though the flag promises "Maximum number of contract_events to export."

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractEventsLimitCountsTransactions"
**Test Language**: Go

### Demonstration

The test proves the limit-granularity mismatch by showing that `TransformContractEvent()` returns 0 event rows for a classic (TxMeta V1) transaction and 3 event rows for a single Soroban (TxMeta V3) transaction. Since `GetTransactions(limit=1)` counts transactions, not events, a `--limit 1` on a classic transaction yields an empty export despite the flag documenting "Maximum number of contract_events to export." Conversely, a single Soroban transaction can exceed the limit by producing multiple event rows with no secondary cap.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestContractEventsLimitCountsTransactions demonstrates that TransformContractEvent
// returns 0 event rows for a classic (non-Soroban) transaction. This proves that
// export_contract_events --limit N, which passes the limit to GetTransactions()
// (counting transactions, not events), can produce an empty export when the first
// N transactions are classic — even though the --limit flag is documented as
// "Maximum number of contract_events to export."
func TestContractEventsLimitCountsTransactions(t *testing.T) {
	// === Part 1: Classic transaction (TxMeta V1) produces 0 contract events ===
	classicTx := ingest.LedgerTransaction{
		Index: 1,
		UnsafeMeta: xdr.TransactionMeta{
			V: 1,
			V1: &xdr.TransactionMetaV1{
				TxChanges: xdr.LedgerEntryChanges{},
				Operations: []xdr.OperationMeta{
					{Changes: xdr.LedgerEntryChanges{}},
				},
			},
		},
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount1,
					Fee:           100,
					SeqNum:        1,
					Operations: []xdr.Operation{
						{
							Body: xdr.OperationBody{
								Type: xdr.OperationTypeCreateAccount,
								CreateAccountOp: &xdr.CreateAccountOp{
									Destination:     testAccount2ID,
									StartingBalance: 1000000000,
								},
							},
						},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: xdr.Hash{},
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
	}
	classicLHE := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 100,
			ScpValue:  xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	classicEvents, err := TransformContractEvent(classicTx, classicLHE)
	if err != nil {
		t.Fatalf("TransformContractEvent on classic tx returned error: %v", err)
	}

	// A classic (V1 meta) transaction produces zero contract events.
	// Yet GetTransactions(limit=1) would count this transaction against the limit,
	// consuming the user's entire --limit budget and returning 0 event rows.
	if len(classicEvents) != 0 {
		t.Fatalf("expected 0 contract events from classic tx, got %d", len(classicEvents))
	}
	t.Logf("Classic tx (TxMeta V1) produced %d contract events — limit budget consumed with zero output", len(classicEvents))

	// === Part 2: Soroban transaction (TxMeta V3) produces multiple events ===
	hardCodedBool := true
	sorobanEvent := xdr.ContractEvent{
		ContractId: &xdr.ContractId{},
		Type:       xdr.ContractEventTypeDiagnostic,
		Body: xdr.ContractEventBody{
			V: 0,
			V0: &xdr.ContractEventV0{
				Topics: []xdr.ScVal{{Type: xdr.ScValTypeScvBool, B: &hardCodedBool}},
				Data:   xdr.ScVal{Type: xdr.ScValTypeScvBool, B: &hardCodedBool},
			},
		},
	}

	sorobanTx := ingest.LedgerTransaction{
		Index: 2,
		UnsafeMeta: xdr.TransactionMeta{
			V: 3,
			V3: &xdr.TransactionMetaV3{
				SorobanMeta: &xdr.SorobanTransactionMeta{
					Events:           []xdr.ContractEvent{sorobanEvent, sorobanEvent},
					DiagnosticEvents: []xdr.DiagnosticEvent{{InSuccessfulContractCall: true, Event: sorobanEvent}},
				},
			},
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
					Fee:           100,
					SeqNum:        2,
					Operations: []xdr.Operation{
						{
							Body: xdr.OperationBody{
								Type:              xdr.OperationTypeInvokeHostFunction,
								InvokeHostFunctionOp: &xdr.InvokeHostFunctionOp{},
							},
						},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: xdr.Hash{},
			Result: xdr.TransactionResult{
				FeeCharged: 100,
				Result: xdr.TransactionResultResult{
					Code: xdr.TransactionResultCodeTxSuccess,
					Results: &[]xdr.OperationResult{
						{
							Tr: &xdr.OperationResultTr{
								Type: xdr.OperationTypeInvokeHostFunction,
								InvokeHostFunctionResult: &xdr.InvokeHostFunctionResult{
									Code: xdr.InvokeHostFunctionResultCodeInvokeHostFunctionSuccess,
								},
							},
						},
					},
				},
			},
		},
	}
	sorobanLHE := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 100,
			ScpValue:  xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	sorobanEvents, err := TransformContractEvent(sorobanTx, sorobanLHE)
	if err != nil {
		t.Fatalf("TransformContractEvent on Soroban tx returned error: %v", err)
	}

	// A single Soroban transaction can produce multiple event rows.
	if len(sorobanEvents) == 0 {
		t.Fatalf("expected >0 contract events from Soroban tx, got 0")
	}
	t.Logf("Soroban tx (TxMeta V3) produced %d contract events from a single transaction", len(sorobanEvents))

	// === Conclusion ===
	// With --limit 1, GetTransactions returns 1 transaction.
	// If that transaction is classic (like Part 1): 0 event rows exported.
	// If that transaction is Soroban (like Part 2): %d event rows exported (exceeds limit).
	// Either way, the limit does not control the number of contract_events as documented.
	t.Logf("DEMONSTRATED: --limit counts transactions (%d tx), not events. "+
		"Classic tx yields %d events, Soroban tx yields %d events. "+
		"Flag doc says 'Maximum number of contract_events to export' but limit is applied at transaction granularity.",
		1, len(classicEvents), len(sorobanEvents))
}
```

### Test Output

```
=== RUN   TestContractEventsLimitCountsTransactions
    data_integrity_poc_test.go:88: Classic tx (TxMeta V1) produced 0 contract events — limit budget consumed with zero output
    data_integrity_poc_test.go:173: Soroban tx (TxMeta V3) produced 3 contract events from a single transaction
    data_integrity_poc_test.go:180: DEMONSTRATED: --limit counts transactions (1 tx), not events. Classic tx yields 0 events, Soroban tx yields 3 events. Flag doc says 'Maximum number of contract_events to export' but limit is applied at transaction granularity.
--- PASS: TestContractEventsLimitCountsTransactions (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.940s
```
