# H002: Tiny liquidity-pool deposit price bounds collapse to zero

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details.min_price` and `history_operations.details.max_price` for `liquidity_pool_deposit` should preserve the actual nonzero slippage bounds encoded in the operation. If a transaction sets `MinPrice = 1/2147483647` or `MaxPrice = 1/2147483647`, the exported numeric bound should remain nonzero.

## Mechanism

The `liquidity_pool_deposit` branch delegates both bound fields to `addPriceDetails()`, which round-trips `xdr.Price` through `Price.String()` and `strconv.ParseFloat`. Because `Price.String()` is only 7-decimal precision, very small but valid bounds become `"0.0000000"` and the ETL exports `min_price = 0` and/or `max_price = 0`, silently widening the apparent slippage range.

## Trigger

1. Export a successful `liquidity_pool_deposit` whose `MinPrice` or `MaxPrice` is a valid tiny rational such as `1/2147483647`.
2. Inspect `history_operations.details.min_price` / `max_price`.
3. Observe that the numeric field is `0` even though `min_price_r` / `max_price_r` retain the exact rational components.

## Target Code

- `internal/transform/operation.go:addPriceDetails:409-419` — lossy `Price.String()` -> `ParseFloat` conversion
- `internal/transform/operation.go:extractOperationDetails:957-1013` — `liquidity_pool_deposit` writes `min_price` and `max_price` via that helper
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/price.go:String:7-10` — formatter truncates to 7 decimal places

## Evidence

The deposit branch uses `addPriceDetails(details, op.MinPrice, "min")` and `addPriceDetails(details, op.MaxPrice, "max")`, so it inherits the same 7-decimal string truncation as offer operations. Any legitimate bound smaller than `5e-8` is rendered as `"0.0000000"` before parsing, making the exported numeric guardrail look disabled.

## Anti-Evidence

The exact rationals survive separately in `min_price_r` and `max_price_r`, so the raw envelope data is not lost completely. But the primary numeric bound fields that downstream analytics are most likely to read still become wrong.
