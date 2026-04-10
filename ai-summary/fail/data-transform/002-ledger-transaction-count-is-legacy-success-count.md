# H002: Ledger `transaction_count` should include failed transactions

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A naive reading suggests `history_ledgers.transaction_count` should equal the total number of transactions in the ledger, with `successful_transaction_count` and `failed_transaction_count` partitioning that total.

## Mechanism

`extractCounts()` computes `transactionCount = int32(txCount) - failedTxCount`, which makes `transaction_count` identical to `successful_transaction_count` whenever failures occur. If the column were meant to represent total transactions, every ledger with failures would export an understated count.

## Trigger

Export any ledger containing at least one failed transaction and compare `transaction_count` to `successful_transaction_count + failed_transaction_count`.

## Target Code

- `internal/transform/ledger.go:extractCounts:133-165` — derives `transactionCount` as `txCount - failedTxCount`
- `internal/transform/schema.go:LedgerOutput:12-23` — exposes `transaction_count`, `successful_transaction_count`, and `failed_transaction_count` together

## Evidence

The arithmetic in `extractCounts()` does make `transaction_count` collapse to the number of successful transactions. On its face that looks redundant because the successful count is also exported separately.

## Anti-Evidence

`internal/transform/ledger_test.go:119-149` and the checked-in golden ledgers under `testdata/ledgers/*.golden` consistently assert that `transaction_count` equals `successful_transaction_count`, even on ledgers with non-zero `failed_transaction_count`. That broad fixture coverage strongly suggests this is a long-standing schema convention rather than an accidental transform regression.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Repository tests and golden outputs consistently encode `transaction_count` as the successful-transaction count, so there is not enough evidence that the current behavior deviates from the intended exported schema.

### Lesson Learned

Some ledger columns in this ETL use historical BigQuery semantics rather than literal English names. When a suspicious count is asserted across unit tests and golden fixtures, treat it as a likely schema contract unless you can find contrary product documentation.
