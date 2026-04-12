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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (distinct from success/019 which covers the `MaxLedger=0` open-ended formatting bug, not invalid ranges)
**Failed At**: reviewer

### Trace Summary

Traced the full `TransformTransaction()` path in `transaction.go:90-111`. Confirmed the code inconsistency: `time_bounds` (lines 93-97) validates `MaxTime < MinTime && MaxTime != 0` and returns an error, while `ledger_bounds` (lines 107-111) performs no equivalent check. However, the trigger precondition — a transaction with `MinLedger > MaxLedger` (MaxLedger ≠ 0) existing in archived ledger data — is unreachable under the Stellar protocol. Additionally, the proposed "expected behavior" (returning an error) would be harmful in an ETL context.

### Code Paths Examined

- `internal/transform/transaction.go:90-97` — `time_bounds` validation: returns error on `MaxTime < MinTime && MaxTime != 0`
- `internal/transform/transaction.go:107-111` — `ledger_bounds` serialization: formats `[MinLedger,MaxLedger)` with no validation guard
- `internal/transform/schema.go:64` — `LedgerBounds string` field in `TransactionOutput`

### Why It Failed

Three independent reasons:

1. **Unreachable precondition.** Stellar Core validates `LedgerBounds` semantics during transaction validation. A transaction with `MinLedger > MaxLedger` (where `MaxLedger != 0`) is rejected as `txMALFORMED` before inclusion in a ledger. Such transactions never appear in archived ledger data that the ETL processes. The hypothesis's own Anti-Evidence section acknowledges this uncertainty, and the answer is that the trigger data cannot exist in legitimate archives.

2. **Wrong expected behavior.** The hypothesis proposes the ETL should return an error on impossible ledger bounds, consistent with the `time_bounds` validation. But in an ETL context, returning an error aborts the entire transaction export and can halt batch processing. For a data pipeline, faithfully serializing whatever is in the archive (even if semantically invalid) is preferable to crashing on unexpected data. The `time_bounds` validation at lines 93-97 is itself an overly aggressive defensive check that could block exports if edge-case data appeared.

3. **Corrupted archives are out of scope.** The only way malformed `LedgerBounds` could appear in archive data is through archive corruption or compromised validators, both of which are explicitly out of scope per the investigation charter.

### Lesson Learned

Before hypothesizing about missing validation guards in ETL transformation functions, verify that the invalid input data can actually appear in the source (archived ledger data). Stellar Core enforces structural and semantic validity of transaction preconditions before ledger inclusion. Missing defensive checks in the ETL are only bugs if the data they would catch can realistically exist in archives. Additionally, for data pipelines, the correct response to unexpected data is usually faithful serialization (possibly with a warning), not hard errors that abort processing.
