# H001: `min_account_sequence_age` wraps negative above `math.MaxInt64`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a valid transaction that sets `PreconditionsV2.MinSeqAge`, the exported
`history_transactions.min_account_sequence_age` value should preserve the full
unsigned `Duration` from XDR. A transaction with `minSeqAge = 9223372036854775808`
should export that exact value, not a negative number.

## Mechanism

`TransformTransaction()` reads `Envelope.MinSeqAge()` and immediately casts the
returned value to `int64` before storing it in `null.Int`. Upstream XDR defines
`PreconditionsV2.MinSeqAge` as `Duration`, and `Duration` is a `uint64`, so any
valid value above `math.MaxInt64` will wrap into a negative JSON/Parquet export.

## Trigger

Create a classic or fee-bump transaction with `PRECOND_V2.minSeqAge >
9223372036854775807` and run `export_transactions`. The row's
`min_account_sequence_age` will be negative even though the on-chain precondition
is an unsigned duration.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:119-123` — casts `Envelope.MinSeqAge()` to `int64`
- `internal/transform/schema.go:TransactionOutput:65-67` — stores the field as `null.Int`
- `internal/transform/schema_parquet.go:TransactionOutputParquet:56-58` — persists the field as signed `INT64`
- `go-stellar-sdk/xdr/xdr_generated.go:PreconditionsV2:33335-33340` — `PreconditionsV2.MinSeqAge` is `Duration`
- `go-stellar-sdk/xdr/xdr_generated.go:Duration:48765-48768` — `Duration` is `uint64`

## Evidence

The transaction transform already treats `min_account_sequence_age` as an
optional field, but once present it always narrows the XDR value with
`int64(*minSequenceAge)`. The schema and Parquet schema both keep that narrowed
signed representation, so there is no later chance to recover the original
unsigned value.

## Anti-Evidence

Today most real transactions probably use small sequence ages, which may be why
the bug has stayed hidden. But the XDR contract permits the full `uint64` range,
and this path is live for any valid transaction that chooses a large duration.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantially equivalent to fail/data-transform #058 (sequence_time int64 narrowing)
**Failed At**: reviewer

### Trace Summary

Traced `MinSeqAge()` from the XDR SDK through `TransformTransaction()` to the output schema. The XDR type chain is `PreconditionsV2.MinSeqAge` → `Duration` → `Uint64` → `uint64`. The transform casts this to `int64` at line 122 via `null.IntFrom(int64(*minSequenceAge))`. The narrowing is real — values above `math.MaxInt64` would wrap negative. However, `Duration` represents seconds, and `math.MaxInt64` seconds ≈ 292 billion years, far exceeding any physically meaningful duration.

### Code Paths Examined

- `internal/transform/transaction.go:119-123` — Confirmed `int64(*minSequenceAge)` narrowing from `*Duration` (uint64)
- `internal/transform/schema.go:66` — `MinAccountSequenceAge` is `null.Int` (int64-backed)
- `internal/transform/schema_parquet.go:57` — Parquet field is signed `INT64`
- `go-stellar-sdk/xdr/transaction.go:46-54` — `MinSeqAge()` returns `*Duration`, pointing to `&tx.Cond.V2.MinSeqAge`
- `go-stellar-sdk/xdr/xdr_generated.go:48768` — `type Duration Uint64` (uint64)
- Horizon reference: `transaction_batch_insert_builder.go:244-249` — Horizon stores Duration as `null.String` via `fmt.Sprint(uint64(*d))`, explicitly preserving the unsigned value

### Why It Failed

This is the same uint64→int64 narrowing pattern already rejected in fail #058 (`sequence_time`). The `Duration` type represents seconds; reaching `math.MaxInt64` requires ~292 billion years — a value that cannot appear in any legitimate Stellar ledger. While Horizon sidesteps the narrowing by using string representation, the stellar-etl's int64 choice is a deliberate schema design that is safe within the domain's value range. The trigger condition is purely theoretical and does not meet the Critical/High/Medium severity threshold.

### Lesson Learned

Duration and timestamp fields backed by XDR `uint64` types share the same characteristic: the domain semantics (seconds since epoch or duration in seconds) constrain values far below `math.MaxInt64`. The lesson from #058 applies identically — verify that the domain can realistically reach the dangerous range before treating a uint64→int64 narrowing as actionable.
