# H057: Liquidity-pool `rounding_slippage` sentinel values are a transform bug

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If liquidity-pool rounding slippage cannot be computed for a trade, the exporter
should emit `NULL` or fail explicitly instead of materializing sentinel values
such as `math.MaxInt64` or `math.MinInt64` as though they were real basis-point
 measurements.

## Mechanism

`roundingSlippage()` substitutes `math.MaxInt64` / `math.MinInt64` whenever
`orderbook.CalculatePoolPayout(..., true)` returns `ok=false`. At first glance
this looks like a silent data-corruption path because valid-looking trade rows can
carry absurd slippage numbers instead of "unavailable."

## Trigger

Export a path-payment liquidity-pool trade whose reserves and amounts trigger the
overflow workaround path in `CalculatePoolPayout()`. The resulting row will carry
an extreme sentinel in `rounding_slippage`.

## Target Code

- `internal/transform/trade.go:roundingSlippage:363-394` — substitutes `math.MaxInt64` / `math.MinInt64`
- `go-stellar-sdk/processors/trade/trade.go:roundingSlippage:393-420` — upstream trade processor uses the same workaround

## Evidence

The local transform really does emit sentinel integers instead of nulls on the
`!ok` path, and upstream orderbook tests show positive inputs can drive overflow
handling. So the suspicious behavior is real, not dead code.

## Anti-Evidence

The same workaround appears verbatim in the upstream `stellar/go` trade processor
that this project mirrors. That makes this behavior part of the upstream SDK's
current trade-processing contract rather than a transform-specific divergence.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This path is inherited upstream behavior, not a local ETL-only corruption bug.
The transform matches the canonical `stellar/go` trade processor's sentinel
workaround exactly, so the issue belongs to the upstream SDK / ecosystem
convention and is out of scope for this pass.

### Lesson Learned

When a suspicious transform branch is copied verbatim from the upstream
`stellar/go` processor, treat it as an SDK-contract question first, not as a new
ETL bug. Mirror behavior is only viable if the ETL diverges from upstream or adds
its own extra corruption on top.
