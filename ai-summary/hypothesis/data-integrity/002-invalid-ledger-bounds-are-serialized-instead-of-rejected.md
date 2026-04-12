# H002: Invalid Ledger Bounds Are Serialized Instead of Rejected

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a transaction envelope carries malformed `ledger_bounds` with
`MaxLedger < MinLedger` and `MaxLedger != 0`, the exporter should surface that
the precondition itself is invalid instead of serializing it as a valid-looking
range string. At minimum, it should behave consistently with `time_bounds`,
which rejects impossible intervals rather than emitting them.

## Mechanism

`TransformTransaction()` validates `time_bounds` and returns an error when the
upper bound is earlier than the lower bound, but it performs no equivalent
check for `ledger_bounds`. As a result, a malformed precondition like
`MinLedger = 10, MaxLedger = 5` is exported as `"[10,5)"`, which looks like a
real interval even though XDR classifies invalid preconditions as
`txMALFORMED`.

## Trigger

1. Export a ledger range containing a failed transaction whose preconditions
   include `PRECOND_V2.ledgerBounds`.
2. Use a malformed envelope with `ledgerBounds.MinLedger > ledgerBounds.MaxLedger`
   and `ledgerBounds.MaxLedger != 0`.
3. Inspect `history_transactions.ledger_bounds`; the ETL will emit the
   impossible interval string instead of surfacing the invalid precondition.

## Target Code

- `internal/transform/transaction.go:90-110` — validates malformed
  `time_bounds` but serializes malformed `ledger_bounds` verbatim
- `internal/transform/schema.go:64` — exports the invalid range string in the
  `history_transactions.ledger_bounds` column
- `.../go-stellar-sdk/xdr/Stellar-transaction.x:1997-2002` — defines
  `txMALFORMED` for invalid preconditions

## Evidence

The function has a clear precedent for rejecting impossible intervals:
`time_bounds` returns an error when `MaxTime < MinTime && MaxTime != 0`, but
the adjacent `ledger_bounds` branch omits the same guard entirely. XDR also
documents that invalid preconditions are a first-class transaction failure
category, so malformed ledger bounds are not just a formatting curiosity.

## Anti-Evidence

This hypothesis depends on malformed precondition envelopes being present in
archived ledger data. The result-code enum strongly suggests they are
ledger-representable, but I did not construct a PoC transaction here, so
review should confirm reachability against real ledger ingestion.
