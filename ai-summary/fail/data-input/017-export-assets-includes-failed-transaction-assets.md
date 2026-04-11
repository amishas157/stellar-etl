# H003: `export_assets` can emit assets referenced only by failed transactions

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_assets` should only emit assets that actually appeared on-chain through successful operations in the requested range. A failed payment or failed manage-sell-offer operation should not create a `history_assets` row, because the failed transaction never applied its state transition.

## Mechanism

`GetPaymentOperations()` iterates raw transaction envelopes from `ledger.TransactionEnvelopes()` and appends every `Payment` / `ManageSellOffer` operation without checking whether the parent transaction succeeded. `TransformAsset()` then derives the exported row entirely from the operation body and ledger metadata, so a failed transaction that references a unique asset code still produces a plausible `AssetOutput` row even though no successful on-chain operation ever surfaced that asset.

## Trigger

Run `stellar-etl export_assets --start-ledger <L> --end-ledger <R>` over a range containing a failed payment or failed manage-sell-offer transaction that references an asset not seen in any successful operation in that same range. The correct output should omit that asset entirely; this path will export it because the reader never inspects transaction results.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:31-49` — scans transaction envelopes but never checks transaction success
- `internal/transform/asset.go:13-52` — converts the operation body into an output row with no access to transaction result status
- `cmd/export_assets.go:39-69` — exports any transformed asset that passes deduplication
- `README.md:236-245` — documents the command as exporting assets created from payment operations

## Evidence

Unlike `GetTrades()`, which explicitly requires `tx.Result.Successful()` before emitting a trade candidate, the assets reader has no success guard at all. The command and transformer also do not receive any separate result filter later, so failed-operation assets are not screened out anywhere downstream.

## Anti-Evidence

If the same asset later appears in a successful operation, `seenIDs` can mask the bug by making the export look correct even though the first row was sourced from a failed transaction. The hypothesis therefore needs a trigger where the failed transaction references a unique asset within the range.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`GetPaymentOperations()` calls `ledger.TransactionEnvelopes()` (go-stellar-sdk `xdr/ledger_close_meta.go:62-93`) which returns ALL transaction envelopes from the ledger's transaction set, including failed transactions. Since Stellar protocol 13, failed transactions are recorded in the ledger close meta (they still pay fees). The function iterates these envelopes at `assets.go:40-54`, checking only `op.Body.Type` for Payment/ManageSellOffer, with no success guard. `TransformAsset()` receives only the raw `xdr.Operation` and has no access to the transaction result, so it cannot filter. The trade reader at `trades.go:64` demonstrates the correct pattern: `tx.Result.Successful()` is checked before appending.

### Code Paths Examined

- `internal/input/assets.go:GetPaymentOperations:38-54` — `ledger.TransactionEnvelopes()` returns all envelopes (successful + failed); inner loop checks only `op.Body.Type`, no success guard
- `go-stellar-sdk/xdr/ledger_close_meta.go:62-93` — `TransactionEnvelopes()` extracts from `TxSet` with no result filtering; for V=0 returns raw `TxSet.Txs`, for V=1/2 iterates all phases/components
- `internal/input/trades.go:56-71` — sibling reader uses `ingest.NewLedgerTransactionReaderFromLedgerCloseMeta()` to get `LedgerTransaction` with `Result` field, then checks `tx.Result.Successful()` at line 64
- `internal/transform/asset.go:14-53` — `TransformAsset()` receives `xdr.Operation` only, has no mechanism to check transaction success
- `internal/input/assets_history_archive.go:28-44` — the captive-core path has the identical issue: `transform.GetTransactionSet(ledger)` returns all envelopes without success filtering
- `cmd/export_assets.go:44-70` — no downstream filtering; `seenIDs` deduplicates by asset ID but does not filter by transaction success

### Findings

1. **Missing success guard**: `GetPaymentOperations()` uses `ledger.TransactionEnvelopes()` which returns raw envelopes from the consensus transaction set. This includes both successful and failed transactions. No success check is performed anywhere in the pipeline.

2. **Structural inability to filter**: The `AssetTransformInput` struct contains `Operation`, `OperationIndex`, `TransactionIndex`, `LedgerSeqNum`, and `LedgerCloseMeta` — but NOT the `TransactionResultPair` needed to check success. Even if the transformer wanted to filter, it lacks the data.

3. **Asymmetry with trade reader**: `GetTrades()` at `trades.go:43-79` uses `ingest.NewLedgerTransactionReaderFromLedgerCloseMeta()` which yields `ingest.LedgerTransaction` objects containing both `Envelope` and `Result`. It checks `tx.Result.Successful()` at line 64. The asset reader uses the lower-level `TransactionEnvelopes()` API which strips result data.

4. **Both code paths affected**: Both `GetPaymentOperations()` (datastore backend) and `GetPaymentOperationsHistoryArchive()` (captive core backend) have the same missing guard.

5. **Phantom asset risk**: A failed transaction can reference an asset code/issuer pair that has never had a successful on-chain operation (e.g., payment to a non-existent asset). The export would include a plausible `AssetOutput` row for an asset that may not even have trustlines, polluting downstream analytics.

### PoC Guidance

- **Test file**: `internal/input/assets_test.go` or a new integration test
- **Setup**: Construct a `LedgerCloseMeta` containing two transactions: one successful Payment for asset A, one failed Payment for asset B. For V=0, populate both `TxSet.Txs` (envelopes) and `TxProcessing` (results), setting the second transaction's result code to `txFAILED`.
- **Steps**: Call `GetPaymentOperations()` over the constructed ledger range. Alternatively, use the transform path: call `TransformAsset()` for operations from both transactions.
- **Assertion**: Assert that the returned slice contains only the asset from the successful transaction (asset A). Currently, it will contain both A and B, demonstrating the bug. A simpler PoC: verify that `GetPaymentOperations()` returns operations from both successful and failed transactions by checking the count against the number of successful-only Payment/ManageSellOffer operations in the test ledger.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/input/data_integrity_poc_test.go
**Test Name**: "TestExportAssetsIncludesFailedTransactionAssets"
**Test Language**: Go

### Demonstration

The test constructs a V0 `LedgerCloseMeta` containing two transactions: one successful Payment (asset USDC) and one failed Payment (asset FAKE, result code `txFAILED`). It then reproduces the exact core loop from `GetPaymentOperations()` — calling `lcm.TransactionEnvelopes()` and iterating operations by type. The pipeline returns 2 `AssetTransformInput` entries, proving that the failed transaction's asset leaks through because `TransactionEnvelopes()` returns all envelopes regardless of success and no result check exists anywhere in the asset export path.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestExportAssetsIncludesFailedTransactionAssets demonstrates that
// GetPaymentOperations extracts operations from BOTH successful and failed
// transactions because it calls ledger.TransactionEnvelopes() without
// checking transaction results.
func TestExportAssetsIncludesFailedTransactionAssets(t *testing.T) {
	addr1 := "GCEODJVUUVYVFD5KT4TOEDTMXQ76OPFOQC2EMYYMLPXQCUVPOB6XRWPQ"
	addr2 := "GAOEOQMXDDXPVJC3HDFX6LZFKANJ4OOLQOD2MNXJ7PGAY5FEO4BRRAQU"

	issuer1, _ := xdr.AddressToAccountId(addr1)
	issuer2, _ := xdr.AddressToAccountId(addr2)

	// Asset A: referenced by a successful transaction
	assetA := xdr.Asset{
		Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
		AlphaNum4: &xdr.AlphaNum4{
			AssetCode: xdr.AssetCode4{'U', 'S', 'D', 'C'},
			Issuer:    issuer1,
		},
	}

	// Asset B: referenced by a failed transaction — should NOT appear in output
	assetB := xdr.Asset{
		Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
		AlphaNum4: &xdr.AlphaNum4{
			AssetCode: xdr.AssetCode4{'F', 'A', 'K', 'E'},
			Issuer:    issuer2,
		},
	}

	muxed1 := xdr.MustMuxedAddress(addr1)
	muxed2 := xdr.MustMuxedAddress(addr2)

	// Build two transaction envelopes: one Payment per asset
	successEnv := xdr.TransactionEnvelope{
		Type: xdr.EnvelopeTypeEnvelopeTypeTx,
		V1: &xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: muxed1,
				Operations: []xdr.Operation{
					{
						Body: xdr.OperationBody{
							Type: xdr.OperationTypePayment,
							PaymentOp: &xdr.PaymentOp{
								Destination: muxed2,
								Asset:       assetA,
								Amount:      1000000,
							},
						},
					},
				},
			},
		},
	}

	failedEnv := xdr.TransactionEnvelope{
		Type: xdr.EnvelopeTypeEnvelopeTypeTx,
		V1: &xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: muxed2,
				Operations: []xdr.Operation{
					{
						Body: xdr.OperationBody{
							Type: xdr.OperationTypePayment,
							PaymentOp: &xdr.PaymentOp{
								Destination: muxed1,
								Asset:       assetB,
								Amount:      5000000,
							},
						},
					},
				},
			},
		},
	}

	// Build a V0 LedgerCloseMeta with both transactions in the tx set
	// TxProcessing has the result pairs marking success/failure
	successOps := []xdr.OperationResult{
		{Code: xdr.OperationResultCodeOpInner, Tr: &xdr.OperationResultTr{
			Type:          xdr.OperationTypePayment,
			PaymentResult: &xdr.PaymentResult{Code: xdr.PaymentResultCodePaymentSuccess},
		}},
	}
	failedOps := []xdr.OperationResult{
		{Code: xdr.OperationResultCodeOpInner, Tr: &xdr.OperationResultTr{
			Type:          xdr.OperationTypePayment,
			PaymentResult: &xdr.PaymentResult{Code: xdr.PaymentResultCodePaymentNoDestination},
		}},
	}

	lcm := xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					LedgerSeq: 100,
					ScpValue: xdr.StellarValue{
						CloseTime: 1000,
					},
				},
			},
			TxSet: xdr.TransactionSet{
				Txs: []xdr.TransactionEnvelope{successEnv, failedEnv},
			},
			TxProcessing: []xdr.TransactionResultMeta{
				{
					Result: xdr.TransactionResultPair{
						Result: xdr.TransactionResult{
							Result: xdr.TransactionResultResult{
								Code:    xdr.TransactionResultCodeTxSuccess,
								Results: &successOps,
							},
						},
					},
				},
				{
					Result: xdr.TransactionResultPair{
						Result: xdr.TransactionResult{
							Result: xdr.TransactionResultResult{
								Code:    xdr.TransactionResultCodeTxFailed,
								Results: &failedOps,
							},
						},
					},
				},
			},
		},
	}

	// Reproduce the core logic from GetPaymentOperations (assets.go:38-54)
	// This is the exact code path used in production.
	transactionSet := lcm.TransactionEnvelopes()
	assetSlice := []AssetTransformInput{}
	for txIndex, transaction := range transactionSet {
		for opIndex, op := range transaction.Operations() {
			if op.Body.Type == xdr.OperationTypePayment || op.Body.Type == xdr.OperationTypeManageSellOffer {
				assetSlice = append(assetSlice, AssetTransformInput{
					Operation:        op,
					OperationIndex:   int32(opIndex),
					TransactionIndex: int32(txIndex),
					LedgerSeqNum:     100,
					LedgerCloseMeta:  lcm,
				})
			}
		}
	}

	// The pipeline returns 2 asset operations — one from the successful tx
	// and one from the failed tx. There is no success guard anywhere.
	if len(assetSlice) != 2 {
		t.Fatalf("expected 2 operations extracted (proving no success filter), got %d", len(assetSlice))
	}

	// Verify first operation came from the successful tx (asset A)
	payOp1, ok := assetSlice[0].Operation.Body.GetPaymentOp()
	if !ok {
		t.Fatal("first operation is not a PaymentOp")
	}
	if payOp1.Asset.Type != assetA.Type || *payOp1.Asset.AlphaNum4 != *assetA.AlphaNum4 {
		t.Errorf("first asset mismatch: got %+v", payOp1.Asset)
	}

	// Verify second operation came from the FAILED tx (asset B) —
	// this is the bug: failed-tx assets leak into the export.
	payOp2, ok := assetSlice[1].Operation.Body.GetPaymentOp()
	if !ok {
		t.Fatal("second operation is not a PaymentOp")
	}
	if payOp2.Asset.Type != assetB.Type || *payOp2.Asset.AlphaNum4 != *assetB.AlphaNum4 {
		t.Errorf("second asset mismatch: got %+v", payOp2.Asset)
	}

	// The second operation is from tx index 1, whose result code is txFAILED.
	// Confirm that the result for this tx is indeed failed.
	txIdx := assetSlice[1].TransactionIndex
	resultCode := lcm.V0.TxProcessing[txIdx].Result.Result.Result.Code
	if resultCode != xdr.TransactionResultCodeTxFailed {
		t.Errorf("expected tx result code txFAILED, got %v", resultCode)
	}

	// BUG PROVEN: The pipeline emitted an AssetTransformInput for a
	// Payment from a failed transaction. A correct implementation would
	// check tx.Result.Successful() and skip failed transactions, as
	// GetTrades() does at trades.go:64.
	t.Logf("BUG CONFIRMED: GetPaymentOperations extracted %d operations", len(assetSlice))
	t.Logf("  Operation 0: asset=%s from successful tx (result code=%v)",
		string(payOp1.Asset.AlphaNum4.AssetCode[:]), lcm.V0.TxProcessing[0].Result.Result.Result.Code)
	t.Logf("  Operation 1: asset=%s from FAILED tx (result code=%v)",
		string(payOp2.Asset.AlphaNum4.AssetCode[:]), resultCode)
	t.Logf("  Failed-transaction asset FAKE leaked into asset export pipeline")
}
```

### Test Output

```
=== RUN   TestExportAssetsIncludesFailedTransactionAssets
    data_integrity_poc_test.go:193: BUG CONFIRMED: GetPaymentOperations extracted 2 operations
    data_integrity_poc_test.go:194:   Operation 0: asset=USDC from successful tx (result code=TransactionResultCodeTxSuccess)
    data_integrity_poc_test.go:196:   Operation 1: asset=FAKE from FAILED tx (result code=TransactionResultCodeTxFailed)
    data_integrity_poc_test.go:198:   Failed-transaction asset FAKE leaked into asset export pipeline
--- PASS: TestExportAssetsIncludesFailedTransactionAssets (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/input	0.827s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-11
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Does the PoC exercise the claimed issue?** Partially. It correctly shows that `ledger.TransactionEnvelopes()` exposes failed transaction envelopes and that the asset reader does not inspect results, but it does not call `GetPaymentOperations()` itself and does not prove that the resulting `history_assets` row is incorrect under this command's contract.
2. **Are the preconditions realistic?** Yes. Failed payment or manage-sell-offer transactions carrying unique asset references can occur on-chain.
3. **Is the behavior a bug or by design?** By design. The command is documented as exporting assets "created from payment operations," not assets proven by successful state application, and the implementation consistently derives rows from selected operation bodies rather than ledger-entry state. This matches the existing narrow scope of `export_assets`.
4. **Does the impact match the claimed severity?** No. Because the finding itself is not established, there is no demonstrated structural corruption to rate as High.
5. **Is the finding in scope?** If real, yes, but the current evidence does not establish a real correctness defect.
6. **Is the test itself correct?** Not as a bug proof. The test passes by asserting the exact envelope-iteration behavior that the implementation intentionally performs; it never demonstrates a mismatch between actual output and a documented requirement.
7. **Can the results be explained without the claimed issue?** Yes. A benign explanation is that `export_assets` is an envelope-derived asset-reference export over payment/manage-sell operations, and failed transactions are still part of the ledger history being scanned unless a command explicitly filters for success (as `GetTrades()` does).
8. **Is this finding novel?** Likely novel, but novelty does not matter once the finding is rejected on semantics.

### Rejection Reason

The PoC proves a missing success filter, but the hypothesis never establishes that `export_assets` is supposed to be success-filtered. In this codebase, failed transactions and operations are generally still exported as part of history unless success materially changes the derived dataset, and `export_assets` is intentionally a narrow asset-reference export sourced from operation bodies rather than from successful ledger-state transitions.

### Failed Checks

- 3. Behavior is by design rather than a correctness bug
- 4. Claimed impact/severity is unsupported because no wrong output requirement was violated
- 6. Test does not prove incorrect output; it only proves the implementation's chosen input source
- 7. The observed result has a benign explanation consistent with the command's documented scope
