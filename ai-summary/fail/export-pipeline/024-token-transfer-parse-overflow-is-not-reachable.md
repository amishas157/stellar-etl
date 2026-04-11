# H024: Token-transfer amount parsing can overflow to `+Inf` and silently corrupt export rows

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A valid token-transfer row should export a finite numeric `amount` derived from
the event's exact integer payload. The exporter should not turn a legitimate
on-chain amount into `+Inf`, `0`, or a dropped row just because the amount
string passed through `strconv.ParseFloat`.

## Mechanism

`transformEvents()` parses the raw amount string with `strconv.ParseFloat(..., 64)`
and discards the error. I initially suspected that a sufficiently large valid
Soroban token amount could overflow the float parser, leaving `amount` at a
non-finite value while the exporter continued as though the row were fine.

## Trigger

Process token-transfer events whose raw amount is near the upper bound of the
underlying signed amount representation and inspect whether `TransformTokenTransfer`
returns a finite numeric `amount`.

## Target Code

- `internal/transform/token_transfer.go:47-73` — parses raw token amounts with `strconv.ParseFloat(..., 64)` and ignores parse errors
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/contract_events.go:76-82` — upstream token-transfer processor emits fee-event amounts as exact decimal strings from `amount.String128Raw(...)`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:56030-56039` — source amounts are backed by signed `Int128Parts`

## Evidence

The transform undeniably ignores `ParseFloat` errors, which makes overflow or
range failures look scary. If a valid on-chain amount could exceed float64's
finite range, the exporter would have no protection here.

## Anti-Evidence

The upstream processor emits valid decimal strings derived from signed 128-bit
amounts, not arbitrary unbounded integers. A signed 128-bit maximum is on the
order of `1e38`, which is far below float64's finite range (`~1e308`), so valid
token-transfer amounts cannot trigger `ParseFloat` overflow. The real live issue
on this path is already the confirmed precision-loss bug from converting large
exact integers through float64 too early.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Ignoring `ParseFloat` errors is bad hygiene, but overflow-to-`+Inf` is not a
real export trigger for legitimate token-transfer amounts because the source
domain is only signed 128-bit. The reachable corruption remains precision loss,
which is already separately confirmed.

### Lesson Learned

For numeric-parser hypotheses, separate "precision loss inside range" from
"overflow outside range." The first is often real for financial exports; the
second needs a source-domain check before it becomes a viable finding.
