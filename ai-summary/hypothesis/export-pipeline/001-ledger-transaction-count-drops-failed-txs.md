# H001: `transaction_count` drops failed ledger transactions

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Wrong ledger transaction totals
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_ledgers.transaction_count` should report the total number of transactions closed in the ledger. For any ledger row, it should equal `successful_transaction_count + failed_transaction_count`, so a ledger with one success and one failure should export `transaction_count = 2`.

## Mechanism

`extractCounts()` computes `txCount := len(transactions)` and separately tracks `successTxCount` and `failedTxCount`, but then returns `transactionCount = int32(txCount) - failedTxCount`. That makes `transaction_count` duplicate the successful-count path instead of exporting the true ledger transaction total, so every ledger containing failed transactions is undercounted in JSON and Parquet.

## Trigger

Run `export_ledgers` on any ledger that contains at least one failed transaction. The exported row will show `transaction_count` smaller than `successful_transaction_count + failed_transaction_count`.

## Target Code

- `internal/transform/ledger.go:133-165` — `extractCounts()` subtracts failed transactions from the total before returning `transactionCount`
- `internal/transform/ledger.go:104-129` — `TransformLedger()` writes that value into `LedgerOutput.TransactionCount`
- `internal/transform/schema.go:18-21` — the schema exposes distinct `transaction_count`, `successful_transaction_count`, and `failed_transaction_count` columns
- `internal/transform/ledger_test.go:141-149` — the fixture currently expects `transaction_count: 1` alongside `successful_transaction_count: 1` and `failed_transaction_count: 1`

## Evidence

The same function already has the exact total (`len(transactions)`) and the exact failed count, so the correct total is directly available at the export site. The hard-coded ledger test also demonstrates the current output shape: a two-transaction fixture with one failure exports `transaction_count` equal to the successful count, not the total count.

## Anti-Evidence

There is no README table contract spelling out the intended semantics of `transaction_count`, and the current tests lock in the existing behavior. This could be a legacy naming mismatch, but the separate successful/failed columns make the current exported value look redundant rather than intentional.
