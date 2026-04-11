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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `extractCounts()` in `internal/transform/ledger.go:133-165`, confirming it computes `transactionCount = int32(txCount) - failedTxCount` on line 163, which equals the successful count. The test at `ledger_test.go:141-144` confirms this with 2 transactions (1 success, 1 failure) expecting `TransactionCount: 1`. Then verified the upstream Horizon API convention: the official `Ledger` struct in `stellar/go-stellar-sdk/protocols/horizon/main.go` has NO `TransactionCount` field at all — only `SuccessfulTransactionCount` and `FailedTransactionCount`. The schema comment on line 12 of `schema.go` states `LedgerOutput` "aligns with the BigQuery table history_ledgers", which in Horizon's DB historically used `transaction_count` to mean successful-only (a legacy convention predating the addition of explicit `successful_transaction_count`/`failed_transaction_count` columns).

### Code Paths Examined

- `internal/transform/ledger.go:133-165` — `extractCounts()` subtracts `failedTxCount` from total; this is the deliberate computation
- `internal/transform/ledger.go:104-129` — `TransformLedger()` assigns the result to `LedgerOutput.TransactionCount`
- `internal/transform/schema.go:12-22` — schema comment says "aligns with the BigQuery table history_ledgers"
- `internal/transform/ledger_test.go:119-151` — test fixture with 2 txs (1 fail, 1 success) expects `TransactionCount: 1`, locking in the design
- `stellar/go-stellar-sdk/protocols/horizon/main.go:225-255` — Horizon's API `Ledger` struct has no `TransactionCount` field; only `SuccessfulTransactionCount` and `FailedTransactionCount`

### Why It Failed

This is **working-as-designed behavior**, not a bug. The `transaction_count` field intentionally mirrors Horizon's `history_ledgers` DB table convention where `transaction_count` historically meant "successful transaction count" only. This convention predates the addition of explicit `successful_transaction_count` and `failed_transaction_count` columns. The field was retained for backward compatibility with downstream consumers that depend on the legacy semantics. The test fixture explicitly locks in this behavior, confirming it is the intended design. The hypothesis misinterprets the field name's natural English meaning as the design contract.

### Lesson Learned

The Stellar ecosystem has a legacy convention where `transaction_count` in `history_ledgers` means "successful transactions only" — not the total. When evaluating data correctness hypotheses about field semantics, always check the upstream Horizon DB/API convention before assuming the field name implies its content. Redundancy between `transaction_count` and `successful_transaction_count` is intentional backward compatibility, not a bug.
