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
