# H058: Account `sequence_time` can overflow negative through `int64` narrowing

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Low
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`accounts.sequence_time` should preserve the account entry's recorded sequence
time exactly. If the source type is wider than the exported type, the transform
should either preserve the full value or reject impossible inputs instead of
wrapping into a negative integer.

## Mechanism

`TransformAccount()` reads `accountEntry.SeqTime()` and stores it with
`zero.IntFrom(int64(outputSequenceTime))`. Upstream XDR defines `SeqTime()` as a
`TimePoint`, and `TimePoint` is `uint64`, so a sufficiently large value would wrap
negative in the exported row.

## Trigger

Construct an account entry whose `SeqTime` exceeds `9223372036854775807` and run
the account transform. The exported `sequence_time` would become negative.

## Target Code

- `internal/transform/account.go:TransformAccount:54-55` — reads `accountEntry.SeqTime()`
- `internal/transform/account.go:TransformAccount:91-94` — narrows `SeqTime` through `int64(...)`
- `go-stellar-sdk/xdr/account_entry.go:SeqTime:89-100` — `SeqTime()` returns `TimePoint`
- `go-stellar-sdk/xdr/xdr_generated.go:TimePoint:48705-48708` — `TimePoint` is `uint64`

## Evidence

The cast is real and unguarded, and unlike `SequenceNumber` there is no negative
check after narrowing. On paper, this is the same unsigned-to-signed pattern as
the viable `min_account_sequence_age` issue.

## Anti-Evidence

`SeqTime` represents a ledger close time carried through account sequence
precondition state, so legitimate values are ordinary Unix timestamps. Reaching
`math.MaxInt64` would require an absurd far-future chain state, not a realistic
current ledger.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The narrowing exists, but the trigger is only theoretical for real Stellar
ledgers. `SeqTime` tracks timestamp-scale values, so there is no plausible current
ledger that can drive the field above `math.MaxInt64`.

### Lesson Learned

Not every `uint64 -> int64` cast is a live integrity bug. For timestamp-backed
fields, verify that the domain can realistically reach the dangerous range before
treating the narrowing as actionable.
