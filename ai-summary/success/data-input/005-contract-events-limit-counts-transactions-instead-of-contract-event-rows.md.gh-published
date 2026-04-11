# 005: Contract-event limit counts transactions instead of contract-event rows

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness: `--limit` can return empty or oversized contract-event exports
**Subsystem**: data-input
**Final review by**: gpt-5.4, high

## Summary

`export_contract_events --limit N` is documented as the maximum number of contract events to export, but the command spends that budget on source transactions before expanding them into emitted rows. On mainnet ledger `52271339`, running `export_contract_events -s 52271339 -e 52271339 -l 1` logs `attempted_transforms:1` yet writes `2` contract-event rows, proving the limit is applied at the wrong granularity.

## Root Cause

`cmd/export_contract_events.go` passes the caller-visible `limit` directly into `input.GetTransactions()`, and `GetTransactions()` enforces that limit on `len(txSlice)`. `transform.TransformContractEvent()` then expands each returned transaction into the concatenation of transaction events, operation events, and diagnostic events, but `export_contract_events` never applies a secondary cap after that expansion.

## Reproduction

During normal operation, this manifests whenever the first transaction returned by `GetTransactions(limit=N)` emits anything other than exactly one contract-event row. A real pubnet reproduction is:

```bash
./stellar-etl export_contract_events -s 52271339 -e 52271339 -l 1 -o /tmp/contract-events.json
wc -l /tmp/contract-events.json
```

The command succeeds, logs `{"attempted_transforms":1,"failed_transforms":0,"successful_transforms":1}`, and writes `2` JSON rows. The opposite failure mode is also real: if the first limited transaction is classic/non-Soroban, it can consume the whole transaction budget while `TransformContractEvent()` emits `0` rows.

## Affected Code

- `cmd/export_contract_events.go:25-53` — passes `limit` into `GetTransactions()` and then exports every row returned by `TransformContractEvent()`
- `internal/input/transactions.go:23-70` — enforces `limit` on transaction count via `len(txSlice)`
- `internal/transform/contract_events.go:21-67` — expands one transaction into all transaction, operation, and diagnostic event rows
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of `contract_events` to export

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractEventsLimitCountsTransactions`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestContractEventsLimitCountsTransactions(t *testing.T) {
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
	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 100,
			ScpValue:  xdr.StellarValue{CloseTime: 1594272522},
		},
	}

	classicEvents, err := TransformContractEvent(classicTx, lhe)
	if err != nil {
		t.Fatalf("TransformContractEvent(classic) returned error: %v", err)
	}
	if len(classicEvents) != 0 {
		t.Fatalf("classic transaction should emit 0 contract-event rows, got %d", len(classicEvents))
	}

	sorobanTransactions, headers, err := makeContractEventTestInput()
	if err != nil {
		t.Fatalf("makeContractEventTestInput returned error: %v", err)
	}

	sorobanEvents, err := TransformContractEvent(sorobanTransactions[1], headers[1])
	if err != nil {
		t.Fatalf("TransformContractEvent(soroban) returned error: %v", err)
	}
	if len(sorobanEvents) != 3 {
		t.Fatalf("expected the Soroban fixture transaction to emit 3 contract-event rows, got %d", len(sorobanEvents))
	}

	t.Logf("classic transaction rows=%d", len(classicEvents))
	t.Logf("single Soroban transaction rows=%d", len(sorobanEvents))
	t.Log("When export_contract_events passes --limit 1 into GetTransactions(), the budget is consumed by one transaction before TransformContractEvent expands it into 0 or 3 emitted rows.")
}
```

## Expected vs Actual Behavior

- **Expected**: `export_contract_events --limit N` should continue scanning the requested ledger range until it has emitted `N` contract-event rows or exhausted the range.
- **Actual**: `GetTransactions()` stops after `N` transactions, and `export_contract_events` then writes every row from those transactions, so `--limit 1` can emit `0`, `1`, or multiple rows. On ledger `52271339`, it emits `2`.

## Adversarial Review

1. Exercises claimed bug: YES — an independent CLI run on pubnet ledger `52271339` wrote `2` rows while logging only `1` transformed transaction, and the PoC test proves the underlying one-transaction-to-many-rows and one-transaction-to-zero-rows behaviors in the production transform path.
2. Realistic preconditions: YES — classic transactions with no contract events and Soroban transactions with multiple emitted events are both normal ledger contents.
3. Bug vs by-design: BUG — `AddArchiveFlags("contract_events", ...)` defines `--limit` as a contract-event row budget, but the exporter applies it at transaction granularity.
4. Final severity: Medium — this silently drops or overshoots caller-visible batches, but it does not alter financial field values.
5. In scope: YES — this is a concrete production export path that returns wrong-but-plausible output.
6. Test correctness: CORRECT — the PoC uses `TransformContractEvent()` directly on real production types and existing fixtures, and the CLI reproduction removes any alternative explanation.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Apply `limit` to emitted `ContractEventOutput` rows rather than to source transactions. The simplest fix is to track the number of rows written in `export_contract_events.go` and stop once it reaches the requested bound; alternatively, add a contract-event-level reader that enforces the limit after expansion.
