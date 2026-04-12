# H006: Memo Type Uses SDK Enum Names

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Low
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `history_transactions.memo_type` is intended to be a consumer-facing value,
the exporter would ideally emit normalized strings like `text`, `id`, `hash`,
or `return` instead of internal Go enum names.

## Mechanism

`TransformTransaction()` sets `memo_type` to `memoObject.Type.String()`, which
produces values like `MemoTypeMemoText`. That looked suspicious because it
resembles a Go enum identifier rather than a compact output contract.

## Trigger

1. Export any transaction with a text memo.
2. Inspect `history_transactions.memo_type`.
3. Observe that the ETL emits `MemoTypeMemoText`.

## Target Code

- `internal/transform/transaction.go:89` — assigns `memoObject.Type.String()`
- `internal/transform/transaction_test.go:101-102` — test fixtures expect
  `MemoTypeMemoText`
- `.../go-stellar-sdk/ingest/ledger_transaction.go:377-379` — upstream helper
  returns the same enum string

## Evidence

The output is an internal enum name, and the ETL schema stores it as a plain
string with no further normalization.

## Anti-Evidence

The exact same string is emitted by the upstream SDK helper and is asserted by
the ETL's own transaction tests, so this is an established local contract, not
an accidental one-off conversion.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This path is test-covered and intentionally inherited from the upstream ingest
helper. I found no repository contract, documentation, or sibling export that
establishes `text`/`id`/`hash` as the correct ETL output, so this is a naming
convention question rather than demonstrated ETL-side corruption.

### Lesson Learned

SDK-derived string enums are weak integrity candidates when the ETL's own tests
already assert the same values. Without a stronger output contract, internal
enum naming alone is not enough to claim silent data corruption.
