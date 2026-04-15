# H030: `ledger_transaction.tx_ledger_history` is a shared ledger-header blob, not per-transaction history

**Date**: 2026-04-15
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `tx_ledger_history` is meant to capture transaction-specific ledger history, then two different transactions from the same ledger should export different values whenever their envelope/result/meta differ. A raw transaction export field with that name should preserve transaction-scoped history, not a blob that is identical for every row in the ledger.

## Mechanism

`TransformLedgerTransaction()` marshals the caller-provided `xdr.LedgerHeaderHistoryEntry` directly into `tx_ledger_history`. Because the same ledger header is passed for every transaction in the ledger, the exported value is constant across all rows in that ledger and cannot encode any transaction-specific state.

## Trigger

Run `export_ledger_transaction` over any ledger containing at least two transactions and compare the emitted `tx_ledger_history` values across rows. The candidate concern is that the field will repeat exactly even though the transaction envelopes and results differ.

## Target Code

- `internal/transform/ledger_transaction.go:TransformLedgerTransaction:13-57` — marshals `lhe` directly into `TxLedgerHistory`
- `internal/transform/schema.go:LedgerTransactionOutput:86-94` — exposes the field as `tx_ledger_history`

## Evidence

`TransformLedgerTransaction()` derives `TxEnvelope`, `TxResult`, `TxMeta`, and `TxFeeMeta` from the `LedgerTransaction`, but `TxLedgerHistory` comes only from the ledger-wide `xdr.LedgerHeaderHistoryEntry`. There is no transaction-specific transformation step for that column.

## Anti-Evidence

The raw export schema does not define any separate `TransactionHistoryEntry` column, and the code path is internally consistent: the field name is the only thing suggesting per-transaction uniqueness. A recently published raw-export finding already treats `tx_ledger_history` as the place where ledger-header history is stored, which suggests this is the intended contract rather than silent corruption.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The implementation is intentionally exporting the shared `LedgerHeaderHistoryEntry` blob once per transaction row. That may be confusing or redundant, but after tracing the raw schema there is no stronger codebase contract requiring `tx_ledger_history` to vary per transaction, so this is not confirmed data corruption.

### Lesson Learned

For raw XDR tables, a suspiciously named field is not enough by itself; there must be a stronger output contract than "this column carries some ledger-associated history blob." Treat naming ambiguity in the raw exports as a weak signal unless another transform or schema path shows a concrete invariant the value violates.
