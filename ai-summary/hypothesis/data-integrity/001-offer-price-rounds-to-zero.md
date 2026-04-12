# H001: Tiny offer prices collapse to numeric zero in operation details

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details.price` for `manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` should preserve the actual on-chain rational price as a nonzero numeric value whenever `Price.N > 0` and `Price.D > Price.N`. A valid order price like `1/2147483647` should therefore export as approximately `0.0000000004656613`, not `0`.

## Mechanism

`addPriceDetails()` converts the rational `xdr.Price` to a string via `price.String()` and then parses that rounded string back to `float64`. Upstream `xdr.Price.String()` uses `FloatString(7)`, so any legitimate price below `0.00000005` becomes the string `"0.0000000"` before parsing, and the ETL silently exports `details.price = 0` even though `details.price_r` still preserves the exact rational.

## Trigger

1. Export a ledger containing `manage_buy_offer`, `manage_sell_offer`, or `create_passive_sell_offer` with `Price{N:1, D:2147483647}` (or any other valid price below `5e-8`).
2. Inspect `history_operations.details.price`.
3. Observe that the ETL emits `0` while `details.price_r` still reports `{numerator: 1, denominator: 2147483647}`.

## Target Code

- `internal/transform/operation.go:addPriceDetails:409-419` — rounds `xdr.Price` through `price.String()` before `ParseFloat`
- `internal/transform/operation.go:extractOperationDetails:701-749` — live offer-operation branches that feed `details.price`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/price.go:String:7-10` — upstream formatter hard-codes 7 decimal places

## Evidence

`addPriceDetails()` never computes the rational directly from `price.N` and `price.D`; it trusts `price.String()`. The upstream formatter is explicitly `big.NewRat(...).FloatString(7)`, so values smaller than `0.00000005` round to `"0.0000000"` before the ETL parses them back into a numeric zero.

## Anti-Evidence

The exact rational survives in `details.price_r`, so downstream consumers that ignore `details.price` can reconstruct the truth. This does not save the exported numeric `price` field itself, which still becomes wrong while looking perfectly valid.
