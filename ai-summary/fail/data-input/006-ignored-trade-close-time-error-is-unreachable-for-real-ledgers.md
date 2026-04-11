# H006: Ignored Trade Close-Time Conversion Error Could Zero `ledger_closed_at`

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`GetTrades()` should either return a valid ledger close time for every trade input or surface an error to the caller; it should not silently continue with a zero timestamp.

## Mechanism

`GetTrades()` assigns `closeTime, _ := utils.TimePointToUTCTimeStamp(...)`, discarding the returned error. If that conversion could fail for a legitimate ledger, the trade export would stamp rows with Go's zero `time.Time` while appearing otherwise valid.

## Trigger

Export trades from a ledger whose `Header.ScpValue.CloseTime` would cause `TimePointToUTCTimeStamp()` to return an error.

## Target Code

- `internal/input/trades.go:48-49` — discards the close-time conversion error
- `internal/utils/main.go:41-46` — only errors when the converted `int64` timepoint is negative
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:48708` — defines `xdr.TimePoint` as `Uint64`

## Evidence

Ignoring an error on a field that becomes a user-visible timestamp is usually dangerous, and the resulting zero time would be a silent data-quality problem if the error were reachable.

## Anti-Evidence

The only error branch in `TimePointToUTCTimeStamp()` is `intTime < 0`, but `xdr.TimePoint` is an unsigned XDR type and real Stellar close times are ordinary Unix seconds, not values near `math.MaxInt64` that would wrap negative. I did not find a legitimate on-chain ledger that can trigger this path.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The ignored error is effectively unreachable for real ledger data: `xdr.TimePoint` is unsigned, and valid Stellar close times stay far below the `int64` overflow threshold needed to trip the negative check. Without a concrete ledger that can make the conversion fail, this remains a code-smell rather than a real silent-corruption path.

### Lesson Learned

An ignored error only becomes a viable data-integrity finding when the error condition is actually reachable from legitimate chain data. For XDR scalar conversions, check the underlying XDR type and realistic value range before escalating the omission.
