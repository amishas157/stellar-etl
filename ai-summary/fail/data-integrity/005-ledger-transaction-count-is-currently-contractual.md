# H005: Ledger transaction_count should include failed transactions

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_ledgers.transaction_count` would ordinarily be expected to equal the total number of transactions in the ledger, with `successful_transaction_count` and `failed_transaction_count` splitting that total. On a ledger with one success and one failure, the row would therefore export `transaction_count = 2`, `successful_transaction_count = 1`, and `failed_transaction_count = 1`.

## Mechanism

`extractCounts()` computes `transactionCount = int32(txCount) - failedTxCount`, so the exported `transaction_count` duplicates the successful-transaction count instead of the total. That looked like a plausible semantic bug because the same row already exports explicit success and failure counters.

## Trigger

1. Export a ledger that contains both successful and failed transactions.
2. Compare `transaction_count` to `successful_transaction_count + failed_transaction_count`.
3. The current export emits `transaction_count == successful_transaction_count`, not the apparent total.

## Target Code

- `internal/transform/ledger.go:133-165` — ledger counters are derived and `transactionCount` subtracts failures
- `internal/transform/ledger_test.go:141-144` — the unit test explicitly expects `TransactionCount: 1`, `SuccessfulTransactionCount: 1`, `FailedTransactionCount: 1`
- `testdata/ledgers/large_range_ledgers.golden:1` — golden output also shows `transaction_count` duplicated from `successful_transaction_count`

## Evidence

The implementation, unit test, and golden fixtures all agree that `transaction_count` excludes failed transactions. The field name and the presence of separate successful/failed counters made this look like a likely correctness bug at first glance.

## Anti-Evidence

The repository's shipped tests and goldens deliberately encode the current behavior, so this is the documented export contract today rather than an accidental regression. I did not find repository documentation that contradicts those fixtures strongly enough to reclassify the behavior as live corruption from code alone.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`transaction_count` looks semantically surprising, but the implementation is reinforced by both unit-test expectations and checked-in golden files. Without a stronger in-repo contract saying the field must mean total transactions, this is a naming oddity, not a defensible new bug.

### Lesson Learned

When a suspicious semantic mismatch is baked into both tests and goldens, treat it as the repository's current export contract unless a stronger schema/documentation source contradicts it. Redundant counters are not enough by themselves to prove corruption.
