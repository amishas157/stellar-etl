# H003: High-value stroop amounts lose precision across entity tables

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Entity tables that export balances and amounts should preserve the exact 7-decimal value represented by the on-chain stroop integer for every valid Stellar `Int64` amount. For example, `9007199254740993` stroops should export as exactly `900719925.4740993`, and `9223372036854775807` stroops should export as exactly `922337203685.4775807`.

## Mechanism

The transform layer converts core monetary values into `float64` either through `ConvertStroopValueToReal()` or direct `float64(...) / 1e7` division. `float64` cannot exactly represent every 7-decimal Stellar amount once the raw stroop value exceeds `2^53`, so legitimate balances, liabilities, trade amounts, offer amounts, pool reserves, and claimable-balance amounts are rounded to nearby values. Unlike token transfers, these entity outputs usually do not preserve a sibling raw amount field, so the exported value itself becomes the only persisted value and is silently corrupted.

## Trigger

Process any ledger state or trade whose monetary field exceeds `9007199254740992` stroops. Concrete examples:

1. An account or trustline balance of `9007199254740993` stroops should export `900719925.4740993`, but the current path rounds to approximately `900719925.4740992`
2. A valid maximum-sized Stellar amount of `9223372036854775807` stroops should export `922337203685.4775807`, but the current path rounds to approximately `922337203685.4775390625`

## Target Code

- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — converts exact rational stroop values into `float64`
- `internal/transform/account.go:TransformAccount:86-90` — account balances and liabilities
- `internal/transform/trustline.go:TransformTrustline:68-79` — trustline balances and liabilities
- `internal/transform/offer.go:TransformOffer:79-93` — offer amount
- `internal/transform/trade.go:TransformTrade:131-157` — selling and buying amounts
- `internal/transform/liquidity_pool.go:TransformPool:66-82` — pool share count and reserve amounts
- `internal/transform/claimable_balance.go:TransformClaimableBalance:59-67` — claimable-balance amount uses direct `float64(outputAmount) / 1.0e7`
- `internal/transform/schema.go:97-101,160,208,245,272,294,300,524,670` — affected output schemas all persist these values as `float64`

## Evidence

The shared helper is exact only until the final `.Float64()` conversion, after which the decimal is limited by IEEE-754 precision. The same pattern appears across nearly every stateful balance-bearing entity, so the bug is systemic rather than isolated to one table. Numeric spot checks show the problem is not theoretical: `9007199254740993` stroops becomes `900719925.47409915924072265625` after the current conversion path.

## Anti-Evidence

Many everyday Stellar amounts are far below the precision cliff, so a large slice of real traffic will still look correct. The schema has historically used floating columns for these tables, but that legacy choice does not prevent silent corruption on legitimate high-balance rows once the raw stroop value crosses the exact `float64` range.
