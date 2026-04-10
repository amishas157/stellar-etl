# H011: Account sequence extension fields lose nullability in Parquet

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: Low
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `sequence_ledger` or `sequence_time` can be absent independently from an explicit zero value, the exporter should preserve that distinction consistently across JSON and Parquet outputs.

## Mechanism

At first glance, `AccountOutput` looks like another nullable-to-required Parquet bug because it stores `SequenceLedger` and `SequenceTime` as `zero.Int`, while `AccountOutputParquet` uses required `int64` fields. If invalid `zero.Int` values serialized as JSON `null`, Parquet would collapse absent and explicit-zero account sequence-extension values together.

## Trigger

Export account ledger-entry changes where the account sequence extension fields are unset and compare the JSON row to the Parquet row.

## Target Code

- `internal/transform/account.go:54-55` — reads `SeqLedger()` and `SeqTime()`
- `internal/transform/account.go:92-93` — stores them as `zero.Int`
- `internal/transform/schema.go:103-104` — JSON schema uses `zero.Int`
- `internal/transform/schema_parquet.go:83-84` — Parquet schema uses required `int64`
- `internal/transform/parquet_converter.go:112-113` — converter reads `.Int64`

## Evidence

The code pattern resembles the confirmed transaction-nullability issue: a JSON struct uses a nullable-ish wrapper type while the Parquet schema uses required integers. That made it plausible that absent account sequence-extension values would be flattened during Parquet conversion.

## Anti-Evidence

A local check of `github.com/guregu/null/zero.Int` shows `zero.IntFrom(0)` produces `Valid=false` and already marshals to JSON `0`, not `null`. So the JSON layer itself does not preserve an absent-versus-zero distinction here; Parquet is not introducing a new corruption by also emitting `0`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`zero.Int` intentionally serializes invalid values as zero in JSON, so the purported nullability distinction is already gone before the Parquet conversion runs.

### Lesson Learned

Not every wrapper type in `schema.go` is semantically nullable. Before treating a wrapper-to-primitive Parquet conversion as corruption, check whether the wrapper already collapses missing values at the JSON layer.
