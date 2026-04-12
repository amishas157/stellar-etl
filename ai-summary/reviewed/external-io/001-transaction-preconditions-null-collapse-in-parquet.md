# H001: Transaction Parquet collapses absent preconditions into zero values

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a transaction does not set `min_account_sequence`, `min_account_sequence_age`, or `min_account_sequence_ledger_gap`, the Parquet export should preserve those fields as null so downstream analytics can distinguish "precondition absent" from "precondition explicitly set to 0". A transaction that explicitly sets `min_account_sequence_age=0` or `min_account_sequence_ledger_gap=0` should not serialize identically to a transaction that left those preconditions unset.

## Mechanism

`TransformTransaction()` correctly models these preconditions as `null.Int`, but `TransactionOutput.ToParquet()` strips the validity bit by copying `.Int64` into plain `int64` Parquet columns. As a result, absent preconditions are exported as `0`, which silently collides with real zero-valued preconditions and corrupts the semantic meaning of the exported row.

## Trigger

Run `export_transactions --write-parquet` over a range containing both:
1. a transaction with no min-sequence preconditions, and
2. a transaction with `minSeqAge=0` or `minSeqLedgerGap=0`.

The JSON export will distinguish them, but the Parquet rows will both show `0`.

## Target Code

- `internal/transform/transaction.go:113-129` — builds `null.Int` values only when the preconditions are actually present
- `internal/transform/transaction.go:252-254` — stores those nullable values in `TransactionOutput`
- `internal/transform/parquet_converter.go:84-86` — converts invalid `null.Int` values to plain zeroes with `.Int64`
- `internal/transform/schema_parquet.go:56-58` — makes the Parquet columns non-nullable `int64`
- `internal/transform/transaction_test.go:167-168` — existing fixture already exercises explicit zero-valued preconditions

## Evidence

The JSON schema uses `null.Int` specifically for these three fields, which means the transform layer expects absence to be representable. The Parquet converter erases that distinction by reading `.Int64` from the nullable wrapper and writing into non-nullable numeric columns.

## Anti-Evidence

If downstream consumers intentionally treat "missing" and "explicitly zero" as equivalent, the impact is reduced. But the repository already encodes them differently in JSON, which suggests the distinction is meant to be preserved.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The transform layer in `transaction.go:113-129` correctly builds `null.Int{}` (Valid=false, Int64=0) for absent preconditions and `null.IntFrom(value)` (Valid=true, Int64=value) for present ones. The JSON schema in `schema.go:65-67` declares these as `null.Int`, so JSON marshalling correctly emits `null` vs `0`. However, the Parquet converter at `parquet_converter.go:84-86` reads `.Int64` directly from the `null.Int` wrapper, which always yields `0` for absent fields, and the Parquet schema at `schema_parquet.go:56-58` declares these as plain non-nullable `int64` columns (no `repetitiontype=OPTIONAL`). This erases the null/zero distinction.

### Code Paths Examined

- `internal/transform/transaction.go:113-129` — Confirmed: builds `null.Int{}` when precondition is nil, `null.IntFrom(int64(*value))` when present. The `Valid` bit correctly distinguishes absent from set-to-zero.
- `internal/transform/transaction.go:252-254` — Confirmed: stores the `null.Int` values directly in `TransactionOutput`.
- `internal/transform/schema.go:65-67` — Confirmed: fields typed as `null.Int` from `github.com/guregu/null`, which serializes as JSON `null` when `Valid == false`.
- `internal/transform/parquet_converter.go:84-86` — Confirmed: reads `to.MinAccountSequence.Int64`, `to.MinAccountSequenceAge.Int64`, `to.MinAccountSequenceLedgerGap.Int64` — the `.Int64` field access ignores the `.Valid` flag and returns the zero value of the underlying int64.
- `internal/transform/schema_parquet.go:56-58` — Confirmed: all three columns declared as `type=INT64` with no `repetitiontype=OPTIONAL`. The entire Parquet schema only uses `repetitiontype=REPEATED` (for arrays), never `OPTIONAL`.

### Findings

The bug is real and the mechanism is exactly as described. When a transaction has no min-sequence preconditions:
1. `transaction.Envelope.MinSeqNum()` returns `nil` → `outputMinSequence` stays as `null.Int{}` (Valid=false, Int64=0)
2. JSON output: `"min_account_sequence": null` — correct
3. Parquet output: `min_account_sequence = 0` — **incorrect**, indistinguishable from explicitly-set-to-zero

This is a systemic pattern: every `null.Int` field across all schemas loses its validity bit in Parquet conversion. The same issue affects `AccountOutput.SequenceLedger/SequenceTime` (parquet_converter.go:112-113) and `TradeOutput.SellingOfferID/BuyingOfferID/LiquidityPoolFee/RoundingSlippage` (parquet_converter.go:266-272). The transaction precondition fields are the most impactful because `0` is a semantically valid explicit value for age and ledger gap.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go` (append a new test case)
- **Setup**: Create two `LedgerTransformInput` instances:
  1. A transaction with V2 preconditions where `MinSeqAge=0` and `MinSeqLedgerGap=0` are explicitly set
  2. A transaction with V1 or no preconditions (where these fields are absent)
- **Steps**: Call `TransformTransaction()` on both inputs, then call `.ToParquet()` on both outputs
- **Assertion**: Assert that the Parquet output for the absent-precondition transaction produces a different representation than the explicitly-zero transaction. Currently, both produce `MinAccountSequenceAge: 0` and `MinAccountSequenceLedgerGap: 0` in the Parquet struct, demonstrating the null collapse. The JSON outputs (via `json.Marshal`) should show `null` vs `0` respectively, proving the transform layer has the information but the Parquet converter discards it.
