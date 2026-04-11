# H019: LedgerTransaction tx_result includes the result pair intentionally

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `LedgerTransactionOutput.TxResult` were supposed to match `TransactionOutput.TxResult`, it should serialize only the inner `TransactionResultResult` body so the same transaction yields the same base64 result blob in both exports. A raw-ledger-history row would then expose the same logical result payload as the higher-level transaction export.

## Mechanism

`TransformLedgerTransaction()` marshals `&transaction.Result`, while `TransformTransaction()` marshals `&transaction.Result.Result`, so the two outputs diverge for the same underlying transaction. That initially suggests the ledger-transaction export may be wrapping the result with extra data and silently changing the meaning of the `tx_result` column.

## Trigger

Compare `history_transactions.tx_result` and `ledger_transactions.tx_result` for the same fee-bump or classic transaction. The ledger-transaction export includes additional bytes from the full `TransactionResultPair`.

## Target Code

- `internal/transform/ledger_transaction.go:17-24` — ledger-transaction export marshals the full `transaction.Result`
- `internal/transform/transaction.go:54-57` — transaction export marshals only `transaction.Result.Result`
- `internal/transform/ledger_transaction_test.go:46-75` — test fixtures assert the full pair encoding

## Evidence

The two transforms clearly serialize different XDR objects under similarly named `tx_result` fields, and the ledger-transaction test vectors confirm the longer, pair-shaped payload. That makes the difference concrete and reproducible.

## Anti-Evidence

`LedgerTransactionOutput` is a raw history export with a different schema and a dedicated `tx_ledger_history` column, so it does not necessarily promise parity with `TransactionOutput`. The tests lock in the current pair encoding, which strongly suggests the broader payload is intentional.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The ledger-transaction path is intentionally exporting a lower-level history representation, and the tests explicitly anchor `tx_result` to the full `TransactionResultPair`. The mismatch with `TransactionOutput.TxResult` reflects two different output contracts rather than silent corruption inside one contract.

### Lesson Learned

Do not assume similarly named fields across `transform/` entities share identical semantics. Raw-history exports in `ledger_transaction.go` can intentionally preserve more XDR wrapper structure than the normalized transaction export.
