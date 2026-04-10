# H002: Transaction Parquet export collapses nullable preconditions to zero

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`min_account_sequence`, `min_account_sequence_age`, and `min_account_sequence_ledger_gap` should preserve the JSON transform's distinction between “precondition absent” and “precondition explicitly set to 0”. Downstream analytics should be able to tell whether a transaction had no sequence preconditions at all or intentionally used zero-valued ones.

## Mechanism

The JSON schema models all three fields as `null.Int`, and `TransformTransaction()` leaves them invalid unless the envelope actually carries those preconditions. The Parquet schema narrows them to plain `int64`, and `TransactionOutput.ToParquet()` writes `.Int64` directly, so both `null.Int{}` and `null.IntFrom(0)` serialize as `0`. That silently changes the meaning of absent preconditions into present zero-valued preconditions.

## Trigger

Compare two exported transactions: one with no min-sequence preconditions and one with `minSeqAge=0` or `minSeqLedgerGap=0`. Their Parquet rows will both show `0`, even though the JSON transform distinguishes them.

## Target Code

- `internal/transform/transaction.go:113-129` — only populates the fields when the envelope exposes those preconditions
- `internal/transform/schema.go:64-67` — JSON output keeps the three fields nullable
- `internal/transform/schema_parquet.go:54-58` — Parquet schema stores them as non-null `int64`
- `internal/transform/parquet_converter.go:81-86` — converter serializes `.Int64` for all three nullable values
- `internal/transform/transaction_test.go:167-168` — tests already cover a transaction with explicit zero-valued preconditions

## Evidence

`TransformTransaction()` initializes `outputMinSequence`, `outputMinSequenceAge`, and `outputMinSequenceLedgerGap` as empty `null.Int{}` values and only assigns them when the corresponding XDR pointers are non-nil. Yet `TransactionOutput.ToParquet()` writes `to.MinAccountSequence.Int64`, `to.MinAccountSequenceAge.Int64`, and `to.MinAccountSequenceLedgerGap.Int64` into non-null Parquet columns. The test fixture at `transaction_test.go:167-168` proves that `0` is a legitimate explicit value, so collapsing null to `0` loses real semantics.

## Anti-Evidence

If downstream Parquet consumers intentionally treat `0` as a stand-in for null, this may be tolerated operationally. But the JSON schema and transform logic already encode a distinct null state, so the current Parquet conversion is still exporting a wrong value relative to the transform's own contract.
