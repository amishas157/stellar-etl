# H002: Liquidity-pool deposit `min_price` / `max_price` lose sub-1e-7 precision

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_operations` should preserve the exact liquidity-pool deposit price band that bounded execution. If `min_price` or `max_price` is a small but non-zero rational, the exported operation details should remain non-zero and should distinguish neighboring on-chain limits.

## Mechanism

The `liquidity_pool_deposit` branch uses `addPriceDetails()` for both `op.MinPrice` and `op.MaxPrice`. That helper formats the rational with `xdr.Price.String()` and then parses the rounded 7-decimal string into `float64`, so narrow price-band limits below `1e-7` collapse to `0`, and limits that differ only beyond the seventh decimal collapse to the same exported value.

## Trigger

Run `export_operations` over a successful `liquidity_pool_deposit` whose `MinPrice` or `MaxPrice` is something like `xdr.Price{N: 1, D: 2147483647}` or two adjacent prices that differ at the eighth decimal place. The current export will emit identical or zero-valued `details.min_price` / `details.max_price` even though the underlying bounds differ.

## Target Code

- `internal/transform/operation.go:addPriceDetails:409-420` — rounds `xdr.Price` through `price.String()` before export
- `internal/transform/operation.go:extractOperationDetails:1008-1011` — applies that helper to `op.MinPrice` and `op.MaxPrice`

## Evidence

The same helper already feeds offer-operation prices, so the rounding behavior is shared rather than hypothetical. Because the liquidity-pool branch stores `min_price` and `max_price` as the human-facing numeric bounds, any sub-1e-7 price band is exported incorrectly even though the raw XDR price ratio is valid.

## Anti-Evidence

The operation details also include `min_price_r` and `max_price_r`, which preserve the exact rational inputs. Consumers that rebuild the price band from those companion fields can avoid the corruption, but the primary numeric band fields are still wrong.
