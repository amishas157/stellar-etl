# H025: `history_ledgers.transaction_count` should include failed transactions

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

One plausible expectation is that `history_ledgers.transaction_count` should equal the total number of transactions in the ledger, with `successful_transaction_count` and `failed_transaction_count` providing the breakdown. Under that interpretation, a ledger with one success and one failure should export `transaction_count = 2`.

## Mechanism

`extractCounts()` subtracts `failedTxCount` before assigning `transactionCount`, so the field exports only successful transactions even though the schema also has a dedicated `successful_transaction_count` column. That duplication initially looks like a copy-paste or semantic drift bug.

## Trigger

1. Export a ledger containing both successful and failed transactions.
2. Compare `transaction_count`, `successful_transaction_count`, and `failed_transaction_count`.
3. Note that `transaction_count == successful_transaction_count`, not total transactions.

## Target Code

- `internal/transform/ledger.go:133-165` — computes `transactionCount = int32(txCount) - failedTxCount`
- `internal/transform/ledger_test.go:141-145` — test output explicitly expects `TransactionCount: 1` alongside `FailedTransactionCount: 1`
- `testdata/ledgers/single_ledger.golden:1` — golden output also encodes `transaction_count == successful_transaction_count`

## Evidence

The field names are semantically awkward, and the implementation clearly exports successful-only counts in both `transaction_count` and `successful_transaction_count`.

## Anti-Evidence

The repository's tests and goldens intentionally lock in that contract today, so within this codebase it is an established schema behavior rather than an accidental drift that the surrounding tests missed.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

However odd the naming is, the in-repo contract already defines `transaction_count` as the successful-transaction count. The behavior is explicitly asserted by unit tests and checked-in goldens, so this is not an uncovered integrity regression.

### Lesson Learned

When field names are ambiguous, checked-in fixtures matter more than intuition. A suspicious semantic overlap is only a live bug if the repository's own contract does not already codify it.
