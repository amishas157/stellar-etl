# H004: TimePoint signed cast overflow is not realistic for Stellar close times

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: Low
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Ledger close times should export as the UTC timestamp encoded in XDR, and callers should only see an error when the source value is truly invalid for the Stellar domain. Legitimate network ledgers should never receive a fabricated fallback timestamp.

## Mechanism

`TimePointToUTCTimeStamp` casts `xdr.TimePoint` to `int64` and treats negative values as invalid, returning `time.Now()` alongside an error. Since `xdr.TimePoint` is actually a `uint64`, the cast can overflow for values above `math.MaxInt64`, and one caller (`internal/input/trades.go`) ignores the returned error.

## Trigger

Process a ledger header whose `ScpValue.CloseTime` exceeds `9223372036854775807`.

## Target Code

- `internal/utils/main.go:TimePointToUTCTimeStamp:41-46` — casts `uint64` close times to `int64`
- `internal/input/trades.go:GetTrades:48-49` — ignores the helper error and keeps the returned timestamp
- `github.com/stellar/go-stellar-sdk/xdr:TimePoint` — defines the source type as `uint64`

## Evidence

The helper really does cast an unsigned XDR time to `int64`, and the trade input path discards the resulting error. On paper, that could substitute `time.Now()` for the real close time.

## Anti-Evidence

The trigger requires a close time in the year ~292 billion or later, far outside any legitimate Stellar ledger the ETL will process. Real close times are current Unix seconds, not near the signed 64-bit boundary.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Although the cast is technically lossy, the overflow condition is not reachable for legitimate Stellar ledger timestamps, so it does not create a realistic wrong-output path.

### Lesson Learned

Timestamp type mismatches still need domain-range analysis. A theoretical overflow is not a data-integrity finding unless the chain can actually produce values near that boundary.
