# H019: Transaction Parquet collapses absent sequence-precondition fields to `0`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a transaction has no `min_account_sequence`, `min_account_sequence_age`, or `min_account_sequence_ledger_gap` precondition, the Parquet export should preserve that missing state just as the JSON export does. Rows with absent preconditions must remain distinguishable from rows that explicitly set a valid numeric `0` precondition value.

## Mechanism

`TransformTransaction()` models all three sequence-precondition fields as `null.Int` and only sets them when the envelope actually carries the corresponding precondition. `TransactionOutput.ToParquet()` then reads `.Int64` from those nullable wrappers into plain `int64` Parquet columns, so every absent value becomes `0`. That creates a real collision because the repository already treats `0` as a valid concrete value for `min_account_sequence_age` and `min_account_sequence_ledger_gap`.

## Trigger

Run `export_transactions --write-parquet` on any range that contains a transaction without sequence preconditions. Compare the JSON row to the Parquet row, or compare a no-precondition row against a row that explicitly sets `min_account_sequence_age=0` / `min_account_sequence_ledger_gap=0`: JSON preserves the distinction, while Parquet rewrites both to `0`.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:113-129` — builds nullable sequence-precondition fields only when the envelope exposes them
- `internal/transform/schema.go:TransactionOutput:64-67` — models the three fields as `null.Int`
- `internal/transform/schema_parquet.go:TransactionOutputParquet:55-58` — defines the same fields as plain `int64`
- `internal/transform/parquet_converter.go:TransactionOutput.ToParquet:83-86` — flattens invalid `null.Int` values via `.Int64`
- `cmd/export_transactions.go:transactionsCmd.Run:63-65` — writes these lossy Parquet rows during normal exports

## Evidence

The JSON transaction schema explicitly carries `MinAccountSequence`, `MinAccountSequenceAge`, and `MinAccountSequenceLedgerGap` as nullable fields, and `TransformTransaction()` leaves them invalid when the envelope omits those preconditions. The Parquet schema removes that nullability, and the converter reads `.Int64` directly. `internal/transform/transaction_test.go:167-168` already treats `null.IntFrom(0)` as a valid expected output for age and ledger-gap preconditions, so `0` is not a harmless placeholder.

## Anti-Evidence

`min_account_sequence` itself may be less likely to appear as an explicit `0` in live data than the age and ledger-gap fields. But even if only the age/gap fields collide in practice, the Parquet export is still silently converting `null` into a plausible real value.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (same bug class as 013-trade-parquet-null-collapse but distinct fields)

### Trace Summary

`TransformTransaction()` initializes all three precondition fields as `null.Int{}` (zero value: `.Valid=false`, `.Int64=0`) and only populates them via `null.IntFrom()` when the transaction envelope exposes the corresponding precondition. The JSON schema (`TransactionOutput`) preserves nullability through `null.Int`'s JSON marshaling. The Parquet schema (`TransactionOutputParquet`) uses plain `int64`, and `ToParquet()` reads `.Int64` unconditionally, collapsing absent (`null`) values to `0`. The test suite at `transaction_test.go:167-168` uses `null.IntFrom(0)` as a valid expected value, confirming `0` is a legitimate concrete precondition setting.

### Code Paths Examined

- `internal/transform/transaction.go:114,120,126` — `null.Int{}` default initialization for absent preconditions
- `internal/transform/transaction.go:115-117,121-123,127-129` — conditional population via `null.IntFrom()` only when envelope provides values
- `internal/transform/schema.go:65-67` — `null.Int` typed fields in JSON output struct
- `internal/transform/schema_parquet.go:56-58` — plain `int64` typed fields in Parquet output struct (no nullable representation)
- `internal/transform/parquet_converter.go:84-86` — `.Int64` read from `null.Int` without checking `.Valid`
- `internal/transform/transaction_test.go:167-168` — `null.IntFrom(0)` used as valid expected value, proving `0` is a real precondition value

### Findings

The null-to-zero collapse is confirmed for all three transaction precondition fields:
1. **`MinAccountSequence`**: `null.Int{}` → `.Int64` → `0` in Parquet when absent from envelope
2. **`MinAccountSequenceAge`**: Same collapse; test expects `null.IntFrom(0)` as a valid explicit value
3. **`MinAccountSequenceLedgerGap`**: Same collapse; test expects `null.IntFrom(0)` as a valid explicit value

In JSON output, absent preconditions serialize as `null` (via `null.Int`'s marshal behavior), while explicit-zero preconditions serialize as `0`. In Parquet, both become `0`, permanently losing the distinction. This is the same pattern as the confirmed trade-parquet-null-collapse (success/013) but affects a different set of fields.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or `internal/transform/transaction_test.go` if no converter test exists)
- **Setup**: Create two `TransactionOutput` structs: one with `MinAccountSequenceAge: null.Int{}` (absent) and one with `MinAccountSequenceAge: null.IntFrom(0)` (explicit zero)
- **Steps**: Call `.ToParquet()` on both structs
- **Assertion**: Assert that the resulting `TransactionOutputParquet.MinAccountSequenceAge` values are identical (`0` for both), demonstrating the information loss. The JSON marshaled forms should differ (`null` vs `0`).
