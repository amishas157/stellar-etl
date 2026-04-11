# H001: Transaction Parquet collapses nullable V2 precondition zeroes into absent values

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_transactions --write-parquet` should preserve the same distinction as
`TransactionOutput` for `min_account_sequence_age` and
`min_account_sequence_ledger_gap`: transactions without `PRECOND_V2` should
export those fields as null/absent, while transactions that explicitly set
`PRECOND_V2` with `minSeqAge=0` or `minSeqLedgerGap=0` should export literal
`0`.

## Mechanism

`TransformTransaction()` models both fields as `null.Int`, and the checked-in
tests already assert that an explicit V2 zero survives in JSON. But
`TransactionOutputParquet` narrows both columns to plain `int64`, and
`TransactionOutput.ToParquet()` writes `.Int64`, so every non-V2 transaction is
flattened to `0` too. That makes Parquet unable to distinguish "no V2
precondition" from "explicit zero-valued V2 precondition" for two transaction
columns.

## Trigger

Run `export_transactions --write-parquet` over a range containing:

1. a classic transaction with `PRECOND_NONE` or `PRECOND_TIME`, and
2. a transaction with `PRECOND_V2` where `minSeqAge=0` and/or
   `minSeqLedgerGap=0`.

Compare the JSON and Parquet outputs for `min_account_sequence_age` and
`min_account_sequence_ledger_gap`.

## Target Code

- `internal/transform/schema.go:64-68` - JSON schema models both fields as
  nullable `null.Int`
- `internal/transform/transaction.go:119-129` - transformer preserves pointer
  presence from envelope accessors
- `internal/transform/transaction_test.go:165-168` - existing tests assert
  explicit zero-valued V2 preconditions
- `internal/transform/schema_parquet.go:55-59` - Parquet schema makes both
  fields required `int64`
- `internal/transform/parquet_converter.go:83-86` - converter serializes
  `.Int64`, collapsing invalid nulls to `0`

## Evidence

The repository already treats these fields as nullable at the transform layer,
and the fixture in `transaction_test.go` exercises `null.IntFrom(0)` for both
columns. The Parquet path removes that nullability, so non-V2 rows and V2 rows
with explicit zeroes serialize identically.

## Anti-Evidence

A downstream consumer that only cares about the effective constraint might treat
absent and explicit zero as equivalent. If so, the mismatch is representational
rather than a true semantic corruption.

---

## Review

**Verdict**: VIABLE
**Severity**: Low
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4.6, high
**Novelty**: PASS - distinct from the confirmed `min_account_sequence`
nullability issue

### Trace Summary

Tracing the code path shows:

- `Transaction.MinSeqAge()` and `Transaction.MinSeqLedgerGap()` return `nil`
  for non-V2 transactions and `&value` for V2 transactions.
- `TransformTransaction()` preserves that distinction as `null.Int{}` versus
  `null.IntFrom(0)`.
- `TransactionOutput.ToParquet()` then converts both to `0` by reading the
  `.Int64` field directly.

The mechanical JSON/Parquet mismatch is real.

### Findings

This is not the same as the confirmed `min_account_sequence` issue. For
`min_account_sequence`, null and zero mean different protocol constraints. For
`min_account_sequence_age` and `min_account_sequence_ledger_gap`, null means "no
age/gap constraint because the transaction is not using V2" while zero means
"minimum age/gap is zero" - the same effective restriction.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: `internal/transform/data_integrity_poc_test.go`
**Test Name**: `TestParquetCollapsesV2PreconditionZeroes`
**Test Language**: Go

### Demonstration

The PoC constructs:

1. a `PRECOND_TIME` transaction, which transforms to
   `null.Int{Valid:false}` for both fields, and
2. a `PRECOND_V2` transaction with `MinSeqAge=0` and `MinSeqLedgerGap=0`,
   which transforms to `null.IntFrom(0)`.

After `ToParquet()`, both rows contain `0` for both fields, so the test fails
on purpose to show the null/zero encoding collapse.

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-11
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Does the PoC actually exercise the claimed issue?** PASS - yes. The test
   reaches the real production path and shows that `TransactionOutput.ToParquet()`
   collapses `null.Int{Valid:false}` and `null.IntFrom(0)` to the same `int64(0)`.
2. **Are the preconditions realistic?** PASS - yes. Both non-V2 transactions
   and V2 transactions with zero-valued `minSeqAge` / `minSeqLedgerGap` are
   realistic Stellar transaction shapes.
3. **Is the behavior a bug or by design?** FAIL - not as a reportable data
   corruption bug. The upstream XDR definition stores `MinSeqAge` and
   `MinSeqLedgerGap` as inline scalar fields inside `PreconditionsV2`, and for
   these two fields `0` and "not using V2 at all" carry the same effective
   constraint: no minimum age and no minimum ledger gap.
4. **Does the impact match the claimed severity?** FAIL - no. The PoC proves an
   encoding mismatch between JSON and Parquet, but it does not prove that the
   exported Parquet value is wrong for the actual transaction constraint. This
   falls below the reportable Critical/High/Medium data-corruption threshold.
5. **Is the finding in scope?** FAIL - no. The objective is silent data
   corruption that produces materially wrong exported values. Here the Parquet
   output still exports the correct effective value (`0`) for both demonstrated
   cases.
6. **Is the test itself correct?** PASS mechanically - the test is valid for
   showing representation collapse, but its failing assertion assumes "must be
   distinguishable in Parquet" without establishing that the distinction changes
   transaction meaning.
7. **Can the results be explained WITHOUT the claimed issue?** YES - both rows
   show `0` because both demonstrated transactions have the same effective
   minimum age/gap restriction. The lost signal is only whether the transaction
   used the V2 container, not a different constraint value.
8. **Is this finding novel?** PASS - yes, but novelty does not change the lack
   of reportable corruption.

### Rejection Reason

The PoC proves a JSON/Parquet nullability mismatch, but it does not prove
reportable data corruption. Unlike `min_account_sequence`, `null` and `0` are
semantically equivalent for `min_account_sequence_age` and
`min_account_sequence_ledger_gap`, so the Parquet export still conveys the
correct effective precondition.

### Failed Checks

3, 4, 5, 7
