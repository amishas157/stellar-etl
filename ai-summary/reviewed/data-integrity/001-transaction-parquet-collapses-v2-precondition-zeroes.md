# H001: Transaction Parquet collapses nullable V2 precondition zeroes into absent values

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_transactions --write-parquet` should preserve the same distinction as `TransactionOutput` for `min_account_sequence_age` and `min_account_sequence_ledger_gap`: transactions without `PRECOND_V2` should export those fields as null/absent, while transactions that explicitly set `PRECOND_V2` with `minSeqAge=0` or `minSeqLedgerGap=0` should export literal `0`.

## Mechanism

`TransformTransaction()` models both fields as `null.Int`, and the tests already assert that an explicit V2 zero survives in JSON. But `TransactionOutputParquet` narrows both columns to plain `int64`, and `TransactionOutput.ToParquet()` writes `.Int64`, so every non-V2 transaction is flattened to `0` too. That makes Parquet unable to distinguish "no V2 precondition" from "explicit zero-valued V2 precondition" for two separate transaction columns.

## Trigger

Run `export_transactions --write-parquet` over a range containing:
1. a classic transaction with `PRECOND_NONE` or `PRECOND_TIME`, and
2. a transaction with `PRECOND_V2` where `minSeqAge=0` and/or `minSeqLedgerGap=0`.

Compare the JSON and Parquet outputs for `min_account_sequence_age` and `min_account_sequence_ledger_gap`.

## Target Code

- `internal/transform/schema.go:64-68` — JSON schema models both fields as nullable `null.Int`
- `internal/transform/transaction.go:119-129` — transformer preserves pointer presence from envelope accessors
- `internal/transform/transaction_test.go:165-168` — existing tests assert explicit zero-valued V2 preconditions
- `internal/transform/schema_parquet.go:55-59` — Parquet schema makes both fields required `int64`
- `internal/transform/parquet_converter.go:83-86` — converter serializes `.Int64`, collapsing invalid nulls to `0`

## Evidence

The repository already treats these fields as nullable at the transform layer, and the test fixture explicitly exercises `null.IntFrom(0)` for both columns. The Parquet path removes that nullability exactly the same way the already-published `min_account_sequence` bug does, but for the sibling V2 age and ledger-gap fields.

## Anti-Evidence

A downstream consumer that only checks whether the precondition is restrictive might treat absent and explicit zero as equivalent. But the ETL's own schema intentionally represents both columns as nullable, so flattening both cases to `0` is still a structural loss of transaction-precondition state.

---

## Review

**Verdict**: VIABLE
**Severity**: Low
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — related to success/data-transform/003 and fail/data-transform/023 but neither covers these specific fields with correct analysis

### Trace Summary

Traced from the XDR SDK helpers `Transaction.MinSeqAge()` and `Transaction.MinSeqLedgerGap()` through the transform layer to Parquet output. The SDK helpers return `nil` for non-V2 transactions (PRECOND_NONE, PRECOND_TIME) and `&value` for V2. The transform code correctly preserves this as `null.Int{}` (Valid=false) vs `null.IntFrom(0)` (Valid=true). JSON marshaling respects this: `null` vs `0`. The Parquet converter reads `.Int64` directly, which is `0` in both cases, collapsing the distinction.

### Code Paths Examined

- `stellar/go xdr/transaction.go:46-55` — `MinSeqAge()` returns nil for non-V2, `&tx.Cond.V2.MinSeqAge` for V2
- `stellar/go xdr/transaction.go:59-68` — `MinSeqLedgerGap()` same pattern, returns nil for non-V2
- `internal/transform/transaction.go:119-129` — preserves nil→`null.Int{}`, non-nil→`null.IntFrom(int64(*val))`
- `internal/transform/schema.go:66-67` — `MinAccountSequenceAge null.Int`, `MinAccountSequenceLedgerGap null.Int`
- `internal/transform/schema_parquet.go:57-58` — both as plain `int64`, no optionality
- `internal/transform/parquet_converter.go:85-86` — `to.MinAccountSequenceAge.Int64` and `to.MinAccountSequenceLedgerGap.Int64`
- `guregu/null int.go:105-110` — `MarshalJSON` returns `"null"` when `!Valid`, `"0"` when `Valid && Int64==0`

### Findings

The bug is mechanically identical to the confirmed success/data-transform/003 finding for `min_account_sequence`. However, the **semantic impact is much lower**:

- For `min_account_sequence`: null means "source sequence must equal tx.seqNum - 1" (strict), while 0 means "any sequence ≥ 0" (permissive). These have different enforcement semantics. → High severity.
- For `min_account_sequence_age`: null means "no age constraint" and 0 means "minimum age = 0 seconds" — semantically equivalent (both mean no restriction). → Low severity.
- For `min_account_sequence_ledger_gap`: null means "no gap constraint" and 0 means "minimum gap = 0 ledgers" — semantically equivalent. → Low severity.

The structural discrepancy is real (JSON outputs `null`, Parquet outputs `0`), but the practical impact is minimal because the collapsed values carry the same protocol-level meaning. A downstream consumer querying `WHERE min_account_sequence_age IS NOT NULL` to identify V2 transactions would get wrong results in Parquet, but that query pattern is better served by other V2-specific fields.

Note: fail/data-transform/023 previously rejected this finding, but its reasoning was flawed — it conflated "MinSeqAge is not optional within V2" (true) with "the transform cannot preserve a null state" (false — the SDK helpers return nil for non-V2 transactions, and the transform does preserve this).

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go` or `internal/transform/data_integrity_poc_test.go` (if it exists from success 003's PoC)
- **Setup**: Use `makeTransactionTestInput()` to get base transactions; create two variants — one with PRECOND_TIME (non-V2) and one with PRECOND_V2 where MinSeqAge=0 and MinSeqLedgerGap=0
- **Steps**: Run both through `TransformTransaction()`, verify JSON-layer outputs differ (null vs 0), then run `ToParquet()` on both
- **Assertion**: Assert that `parquetNonV2.MinAccountSequenceAge` and `parquetV2.MinAccountSequenceAge` are distinguishable (they won't be — this demonstrates the bug). Same for MinAccountSequenceLedgerGap.
