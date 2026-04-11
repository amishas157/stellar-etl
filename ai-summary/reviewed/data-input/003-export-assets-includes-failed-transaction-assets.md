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
