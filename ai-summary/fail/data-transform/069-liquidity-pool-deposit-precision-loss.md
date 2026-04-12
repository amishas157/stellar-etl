# H001: Liquidity-pool deposit details discard exact deltas through `ParseFloat`

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a successful `liquidity_pool_deposit` operation, the exported
`reserve_a_deposit_amount`, `reserve_b_deposit_amount`, and `shares_received`
detail fields should preserve the exact on-chain decimal values implied by the
pool delta. Two distinct stroop amounts should stay distinct in the JSON
details payload even at large magnitudes.

## Mechanism

`extractOperationDetails()` already computes the exact decimal strings for the
actual deposited reserves and pool-share delta via `amount.String(...)`, but it
immediately round-trips those exact strings through `strconv.ParseFloat(...,
64)`. Once the delta is large enough that `float64` cannot represent adjacent
7-decimal Stellar amounts exactly, distinct pool deposits collapse onto the same
exported numeric value even though the code had an exact representation in hand.

## Trigger

Export `history_operations` for a successful `liquidity_pool_deposit`
transaction whose actual reserve delta or `shares_received` exceeds `float64`
exactness at 7-decimal Stellar precision (for example, two deposits whose exact
stroop amounts differ by `0.0000001` near or above the `2^53` stroop range).
Compare the exported JSON details against the exact `amount.String(...)` value
derived from the operation's liquidity-pool ledger delta.

## Target Code

- `internal/transform/operation.go:974-1019` — computes exact decimal strings from pool deltas, then discards them through `strconv.ParseFloat(...)`
- `internal/transform/operation.go:238-285` — helper derives the exact reserve/share delta from operation-scoped liquidity-pool ledger changes
- `internal/transform/operation_test.go:1757-1803` — tests currently lock these fields in as JSON numbers rather than exact strings

## Evidence

The live deposit branch is one of the few places in this codebase that already
has the exact decimal representation available and then downgrades it anyway.
That matches the repository's previously high-signal pattern of "exact value
computed, then unnecessarily rounded by `ParseFloat`" rather than the broader
float-schema design limit rejected elsewhere.

## Anti-Evidence

Many other amount fields in this repository are also `float64`, so the impact
only manifests on sufficiently large reserve/share deltas. This is still a live
bug because the code already has an exact string and chooses to throw it away.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of success/data-transform/017-liquidity-pool-deposit-detail-float64-rounding.md
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same bug already confirmed and published as success finding 017: liquidity-pool deposit details round large reserve deltas and share amounts through float64 via ParseFloat. The mechanism (exact `amount.String()` converted back to lossy float64 via `strconv.ParseFloat`), the affected fields (`reserve_a_deposit_amount`, `reserve_b_deposit_amount`, `shares_received`), and the target code paths (`extractOperationDetails` LP deposit branch, `getLiquidityPoolAndProductDelta`) are identical.

### Code Paths Examined

- `internal/transform/operation.go:974-1019` — same LP deposit branch identified in success/017
- `internal/transform/operation.go:238-285` — same helper function identified in success/017

### Why It Failed

This is a direct duplicate of the already-confirmed finding `ai-summary/success/data-transform/017-liquidity-pool-deposit-detail-float64-rounding.md.gh-published`, which has already been published. The hypothesis describes the same root cause, same affected fields, same code paths, and same impact.

### Lesson Learned

The LP deposit float64 precision loss pattern has already been fully investigated, confirmed, PoC'd, and published. Future hypothesis generators should check existing success findings before emitting new hypotheses on the same mechanism.
