# H001: Liquidity-pool deposit details discard exact stroop values through `ParseFloat`

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_operations` should preserve the exact realized reserve deltas and pool
shares for successful `liquidity_pool_deposit` operations. If the ledger delta
is `9007199254740993` stroops, the exported detail should preserve the exact
decimal `"900719925.4740993"` rather than rounding it to a nearby binary64
value.

## Mechanism

The live `extractOperationDetails()` path computes exact 7-decimal strings with
`amount.String(...)`, then immediately discards that exact value by
`strconv.ParseFloat(..., 64)` for `reserve_a_deposit_amount`,
`reserve_b_deposit_amount`, and `shares_received`. For large one-stroop-sensitive
amounts, the float64 round-trip changes the exported number: e.g.
`9007199254740993` stroops becomes a float that round-trips to
`9007199254740992` stroops.

## Trigger

1. Export a ledger containing a **successful** `liquidity_pool_deposit`
   operation.
2. Use a deposit whose realized reserve delta or `TotalPoolShares` exceeds
   `2^53` stroops, or otherwise requires exact 7-decimal preservation.
3. Inspect `history_operations.details.reserve_a_deposit_amount`,
   `reserve_b_deposit_amount`, or `shares_received`; the ETL will export the
   rounded float64 approximation instead of the exact decimal already available
   from `amount.String(...)`.

## Target Code

- `internal/transform/operation.go:957-1019` — live
  `liquidity_pool_deposit` detail builder parses exact decimal strings back into
  float64
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/amount/main.go:133-155`
  — `amount.String` produces the exact 7-decimal string form from stroops

## Evidence

The code does not go directly from `xdr.Int64` to float64; it first materializes
the exact decimal with `amount.String(depositedA)`, `amount.String(depositedB)`,
and `amount.String(sharesReceived)`, proving a lossless representation is
already in hand. That matches the previously viable pattern in this repo where
an exact value is computed and then unnecessarily degraded before export.

## Anti-Evidence

The surrounding `details` map historically uses numeric JSON values for several
operation amounts, so reviewers may decide this is an intentional schema choice.
If so, the key question is whether those fields are contractually allowed to be
approximate even though this code path already has the exact decimal available.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `success/utilities/001-float64-stroop-collisions.md.gh-published` and `success/data-integrity/002-token-transfer-float64-rounding.md.gh-published`
**Failed At**: reviewer

### Trace Summary

Traced the LP deposit code path in `internal/transform/operation.go:957-1019`, confirming that `amount.String(depositedA)` produces an exact 7-decimal string which is then parsed to float64 via `strconv.ParseFloat()`. This is functionally identical to `ConvertStroopValueToReal()` — both single-round the exact rational value to the nearest IEEE-754 double, producing the same float64 output for any given stroop input. The LP withdraw path (lines 1051-1059) uses `ConvertStroopValueToReal()` directly, showing an inconsistency in mechanism but not in result.

### Code Paths Examined

- `internal/transform/operation.go:957-984` — LP deposit sets up `depositedA`, `depositedB`, `sharesReceived` as `xdr.Int64` from `getLiquidityPoolAndProductDelta()`
- `internal/transform/operation.go:991-995` — `strconv.ParseFloat(amount.String(depositedA), 64)` → float64 for `reserve_a_deposit_amount`
- `internal/transform/operation.go:1002-1006` — Same pattern for `reserve_b_deposit_amount`
- `internal/transform/operation.go:1015-1019` — Same pattern for `shares_received`
- `internal/transform/operation.go:1051-1059` — LP withdraw uses `ConvertStroopValueToReal()` for the equivalent fields (different mechanism, same result)
- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal` uses `big.NewRat(int64(input), 10000000).Float64()`, producing the same float64 as `strconv.ParseFloat(amount.String(input), 64)`

### Why It Failed

This is a duplicate of two already confirmed and published findings:

1. **`success/utilities/001-float64-stroop-collisions.md.gh-published`** — documents the cross-cutting float64 precision loss for stroop amounts, explicitly mentioning "liquidity-pool reserves" as affected in the adversarial review. The root cause (float64 cannot represent all 7-decimal stroop values) is identical.

2. **`success/data-integrity/002-token-transfer-float64-rounding.md.gh-published`** — documents the `strconv.ParseFloat` pattern specifically, covering the same class of mechanism.

The LP deposit's `amount.String() → strconv.ParseFloat()` path produces a single-rounding result mathematically identical to `big.Rat.Float64()` used in `ConvertStroopValueToReal()`. Both compute the nearest float64 to the exact decimal value. The hypothesis claims the intermediate exact string makes this a different finding, but the exact rational representation exists in `ConvertStroopValueToReal` too (as a `big.Rat`) and is equally discarded. The mechanism differs but the precision loss behavior is identical.

Additionally, `fail/data-integrity/003` (account balances) and `fail/data-integrity/004` (trade amounts) were previously rejected as entity-specific variants of the same cross-cutting float64 precision issue.

### Lesson Learned

The `amount.String() → strconv.ParseFloat()` pattern in LP deposit is a different implementation path from `ConvertStroopValueToReal()`, but produces identical single-rounding float64 precision loss. Having an exact string "in hand" before discarding it is not meaningfully distinct from having an exact `big.Rat` "in hand" before discarding it — both are covered by the existing cross-cutting float64 stroop finding. Entity-specific or mechanism-specific variants of the float64 precision issue should not be resubmitted unless they demonstrate a qualitatively different impact (e.g., double rounding as in token transfers).
