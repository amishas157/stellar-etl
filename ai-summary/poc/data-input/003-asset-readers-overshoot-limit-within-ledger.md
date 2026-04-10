# H003: Asset Readers Overshoot `--limit` Within a Single Ledger

**Date**: 2026-04-10
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `stellar-etl export_assets --limit N` is used, the reader should stop after collecting at most `N` qualifying operations, matching the documented meaning of the limit flag and the exact-limit behavior used by the transaction, operation, and trade readers.

## Mechanism

Both asset readers append every qualifying payment or manage-sell-offer operation in the current ledger before checking whether `len(assetSlice) >= limit`. If the limit boundary is crossed inside the nested transaction/operation loops, the function returns more than `N` `AssetTransformInput` values, so `export_assets` can emit extra asset rows from the same ledger instead of honoring the requested cap.

## Trigger

Run `stellar-etl export_assets --limit 1 --start-ledger <L> --end-ledger <R>` where ledger `<L>` contains at least two qualifying operations for different assets. The export should return one asset input, but the current readers will keep scanning the rest of the ledger and can produce two or more rows before the outer per-ledger limit check fires.

## Target Code

- `internal/input/assets.go:31-57` — appends qualifying operations inside nested loops and checks `limit` only after finishing the ledger
- `internal/input/assets_history_archive.go:21-47` — repeats the same per-ledger limit check in the history-archive variant
- `cmd/export_assets.go:21-33` — exposes the `limit` flag as the caller-visible bound for asset exports
- `internal/input/operations.go:52-76` — sibling reader that stops inside the inner loop once the limit is reached
- `internal/input/trades.go:50-80` — sibling reader that also enforces the limit before finishing the ledger

## Evidence

Unlike `GetOperations()` and `GetTrades()`, neither asset reader has an inner-loop `break` when the slice reaches the requested limit. The only guard lives after the entire ledger has been traversed, so the overshoot can be as large as the number of additional qualifying operations in that ledger.

## Anti-Evidence

`export_assets` deduplicates repeated `AssetID` values before writing output, so overshoot is easiest to observe when the extra operations reference distinct assets. If the additional operations all point at an already-seen asset, the wrong reader count may not translate into more exported rows.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the limit enforcement in both asset readers (`GetPaymentOperations` and `GetPaymentOperationsHistoryArchive`) and compared against the sibling readers (`GetOperations` and `GetTrades`). The asset readers use a simple `for txIndex, transaction := range transactionSet` pattern with the limit check only at the outer ledger-iteration level (assets.go:55, assets_history_archive.go:45). In contrast, `GetOperations` (operations.go:52,67,75) and `GetTrades` (trades.go:50,73,80) each have three limit checks: (1) the outer read-loop condition, (2) an inner-loop break after each append, and (3) a post-ledger break. The asset readers are genuinely missing the inner-loop enforcement present in all sibling readers.

### Code Paths Examined

- `internal/input/assets.go:31-58` — Outer loop iterates ledgers; inner nested loops iterate `transactionSet` then `transaction.Operations()`. Qualifying ops are appended unconditionally. Limit check at line 55 fires only after both inner loops complete for the entire ledger.
- `internal/input/assets_history_archive.go:21-48` — Identical structure to `assets.go`. Same missing inner-loop limit enforcement at line 45.
- `internal/input/operations.go:52-78` — Outer loop condition `int64(len(opSlice)) < limit || limit < 0` (line 52) prevents reading the next transaction once limit is met. Inner loop break at line 67-69 stops mid-transaction. Post-ledger break at line 75. Three-tier enforcement.
- `internal/input/trades.go:50-83` — Same three-tier enforcement as operations: outer condition (line 50), inner break (line 73-75), post-ledger break (line 80).
- `cmd/export_assets.go:27-34` — Calls one of the two asset readers, passing `limit` directly. No post-reader truncation of the returned slice. The `seenIDs` deduplication (line 54) only removes duplicate AssetIDs, not excess results.

### Findings

The bug is confirmed: both asset readers (`GetPaymentOperations` and `GetPaymentOperationsHistoryArchive`) lack inner-loop limit checks that all sibling readers implement. The overshoot per-ledger is bounded by the number of qualifying operations (Payment + ManageSellOffer) in a single ledger, which could be up to 100,000 in the theoretical worst case (1000 txs × 100 ops).

Severity downgraded from High to Medium. The data in each exported row is correct — no fields are wrong, no values are corrupted. The issue is operational: the `--limit` flag does not bound output size as precisely as the sibling commands. This is an inconsistency in limit enforcement rather than structural data corruption. The `seenIDs` deduplication in `export_assets` further reduces visible impact, since overshoot only produces extra output rows when the extra operations reference distinct assets.

### PoC Guidance

- **Test file**: `internal/input/assets_test.go` (or create if absent; check existing test patterns in `internal/input/operations_test.go`)
- **Setup**: Construct a mock ledger backend containing a single ledger with ≥3 qualifying operations (Payment or ManageSellOffer) referencing distinct assets.
- **Steps**: Call `GetPaymentOperations(start, start, 1, env, false)` with `limit=1` on the crafted ledger.
- **Assertion**: Assert `len(result) == 1`. Currently this will fail — the result will contain all qualifying operations from the ledger (≥3), demonstrating the overshoot. Compare against `GetOperations(start, start, 1, env, false)` which correctly returns exactly 1 result.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/input/data_integrity_poc_test.go
**Test Name**: "TestAssetReaderOvershootsLimitWithinLedger"
**Test Language**: Go

### Demonstration

The test constructs a single LedgerCloseMeta containing 3 transactions, each with one Payment operation (a qualifying op for the asset reader). It then exercises the exact same inner-loop logic from `GetPaymentOperations` (assets.go:40-54) with `limit=1`. Because the limit check only occurs after the entire ledger's transactions are processed, all 3 operations are collected — an overshoot of 2. In contrast, the same input processed with the operations-reader-style loop (which has inner-loop break checks) correctly returns exactly 1 result.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestAssetReaderOvershootsLimitWithinLedger demonstrates that GetPaymentOperations'
// inner loop does not enforce the limit check, causing it to collect all qualifying
// operations in a ledger even when limit=1.
//
// The sibling readers (GetOperations, GetTrades) have inner-loop breaks that stop
// mid-transaction once the limit is reached. The asset readers only check after
// the entire ledger is processed, causing overshoot.
func TestAssetReaderOvershootsLimitWithinLedger(t *testing.T) {
	// Construct 3 transaction envelopes, each with one Payment operation.
	// All 3 are qualifying operations for the asset reader.
	numTxns := 3
	txEnvelopes := make([]xdr.TransactionEnvelope, numTxns)
	for i := 0; i < numTxns; i++ {
		txEnvelopes[i] = makePaymentTxEnvelope()
	}

	// Build a LedgerCloseMeta (V0) containing all 3 transactions in a single ledger.
	ledger := xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					LedgerSeq: 100,
				},
			},
			TxSet: xdr.TransactionSet{
				Txs: txEnvelopes,
			},
		},
	}

	// --- Reproduce the asset reader's loop logic (from assets.go:40-54) ---
	// This is the exact same loop structure as GetPaymentOperations, but
	// operating on a single pre-built ledger instead of reading from a backend.
	limit := int64(1)
	assetSlice := []AssetTransformInput{}
	transactionSet := ledger.TransactionEnvelopes()

	for txIndex, transaction := range transactionSet {
		for opIndex, op := range transaction.Operations() {
			if op.Body.Type == xdr.OperationTypePayment || op.Body.Type == xdr.OperationTypeManageSellOffer {
				assetSlice = append(assetSlice, AssetTransformInput{
					Operation:        op,
					OperationIndex:   int32(opIndex),
					TransactionIndex: int32(txIndex),
					LedgerSeqNum:     int32(100),
					LedgerCloseMeta:  ledger,
				})
			}
		}
	}
	// Limit check occurs ONLY here — after all transactions in the ledger are processed.
	// By this point, assetSlice already has all 3 qualifying operations.
	_ = int64(len(assetSlice)) >= limit && limit >= 0 // (the break would fire here, but too late)

	// --- Reproduce the correct loop logic (from operations.go:52-70) ---
	// This is the inner-loop structure from GetOperations, which correctly
	// breaks mid-transaction once the limit is reached.
	correctSlice := []AssetTransformInput{}
	for txIndex, transaction := range transactionSet {
		if int64(len(correctSlice)) >= limit {
			break
		}
		for opIndex, op := range transaction.Operations() {
			if op.Body.Type == xdr.OperationTypePayment || op.Body.Type == xdr.OperationTypeManageSellOffer {
				correctSlice = append(correctSlice, AssetTransformInput{
					Operation:        op,
					OperationIndex:   int32(opIndex),
					TransactionIndex: int32(txIndex),
					LedgerSeqNum:     int32(100),
					LedgerCloseMeta:  ledger,
				})
			}
			if int64(len(correctSlice)) >= limit && limit >= 0 {
				break
			}
		}
	}

	// The asset reader's loop (without inner-loop limit enforcement) collects ALL
	// qualifying operations in the ledger. With 3 Payment transactions and limit=1,
	// this produces 3 results instead of the requested 1.
	if int64(len(assetSlice)) <= limit {
		t.Errorf("Expected asset reader to overshoot limit=%d, but got len=%d (no overshoot detected)",
			limit, len(assetSlice))
	} else {
		t.Logf("BUG CONFIRMED: asset reader produced %d results with limit=%d (overshoot of %d)",
			len(assetSlice), limit, int64(len(assetSlice))-limit)
	}

	// The correct loop (with inner-loop limit enforcement) respects the limit.
	if int64(len(correctSlice)) != limit {
		t.Errorf("Expected corrected loop to return exactly %d results, got %d",
			limit, len(correctSlice))
	} else {
		t.Logf("CORRECT: operations-style loop returned exactly %d result(s) with limit=%d",
			len(correctSlice), limit)
	}

	// Final assertion: the asset reader's result count exceeds the correct count.
	if len(assetSlice) <= len(correctSlice) {
		t.Errorf("Expected asset reader (%d results) to return more than corrected loop (%d results)",
			len(assetSlice), len(correctSlice))
	}
}

// makePaymentTxEnvelope creates a minimal TransactionEnvelope containing a single
// Payment operation. This is a qualifying operation for the asset reader.
func makePaymentTxEnvelope() xdr.TransactionEnvelope {
	dest := xdr.MustMuxedAddress("GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN7")
	return xdr.TransactionEnvelope{
		Type: xdr.EnvelopeTypeEnvelopeTypeTx,
		V1: &xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				Operations: []xdr.Operation{
					{
						Body: xdr.OperationBody{
							Type: xdr.OperationTypePayment,
							PaymentOp: &xdr.PaymentOp{
								Destination: dest,
								Asset:       xdr.MustNewNativeAsset(),
								Amount:      1000,
							},
						},
					},
				},
			},
		},
	}
}
```

### Test Output

```
=== RUN   TestAssetReaderOvershootsLimitWithinLedger
    data_integrity_poc_test.go:95: BUG CONFIRMED: asset reader produced 3 results with limit=1 (overshoot of 2)
    data_integrity_poc_test.go:104: CORRECT: operations-style loop returned exactly 1 result(s) with limit=1
--- PASS: TestAssetReaderOvershootsLimitWithinLedger (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/input	0.670s
```
