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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete code path from `extractOperationDetails` LP deposit branch (operation.go:957-1013) through `addPriceDetails` (operation.go:409-421) into `xdr.Price.String()` (go-stellar-sdk xdr/price.go:8-10). Confirmed that `Price.String()` delegates to `big.NewRat(N, D).FloatString(7)`, which produces exactly 7 decimal places. Verified with actual Go execution that `Price{1, 20000001}` produces `"0.0000000"` → `ParseFloat` returns `0.0`. The same `addPriceDetails` function is also called for ManageBuyOffer (line 709), ManageSellOffer (line 728), and CreatePassiveSellOffer (line 746) prices, making this a cross-operation issue.

### Code Paths Examined

- `internal/transform/operation.go:409-421` — `addPriceDetails()` calls `price.String()` then `strconv.ParseFloat(s, 64)`, assigning result to `prefix+"price"`. Also sets `prefix+"price_r"` with exact N/D.
- `internal/transform/operation.go:1008-1012` — LP deposit calls `addPriceDetails(details, op.MinPrice, "min")` and `addPriceDetails(details, op.MaxPrice, "max")`
- `go-stellar-sdk xdr/price.go:8-10` — `Price.String()` returns `big.NewRat(int64(p.N), int64(p.D)).FloatString(7)` — exactly 7 decimal places
- `internal/transform/operation.go:709,728,746` — same `addPriceDetails` used for ManageBuyOffer, ManageSellOffer, CreatePassiveSellOffer prices

### Findings

**Confirmed**: Any `xdr.Price` where the rational value N/D < 5×10⁻⁸ (approximately D > 20,000,000×N) will be exported as `0.0` in the scalar price field while the companion `*_price_r` rational preserves the exact value.

**Go verification**: Tested with `big.NewRat(1, 20000001).FloatString(7)` → `"0.0000000"` → `ParseFloat` → `0.0`. Boundary confirmed: `Price{1, 20000000}` → `"0.0000001"` (survives), `Price{1, 20000001}` → `"0.0000000"` (truncated to zero).

**Scope**: Affects 5 call sites — LP deposit min_price, LP deposit max_price, ManageBuyOffer price, ManageSellOffer price, CreatePassiveSellOffer price. All share the same `addPriceDetails` code path.

**Severity rationale**: Downgraded from Critical to Medium because: (1) affected prices are extremely small (< 5×10⁻⁸), limiting real-world prevalence; (2) exact rational components survive in companion `*_price_r` fields; (3) for LP deposit slippage bounds, a price this small is functionally near-zero. However, the bug is real — a nonzero financial field becomes exactly zero, which is semantically incorrect and could mislead downstream analytics that read only the scalar price field.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Create a `LiquidityPoolDepositOp` with `MinPrice: xdr.Price{N: 1, D: 20000001}` and `MaxPrice: xdr.Price{N: 1, D: 2147483647}`. Wrap in a successful transaction with appropriate LP metadata.
- **Steps**: Call `TransformOperation()` on the constructed operation. Extract `details["min_price"]` and `details["max_price"]` from the result.
- **Assertion**: Assert that `details["min_price"]` equals `0.0` (demonstrating the bug). Also assert that `details["min_price_r"]` correctly preserves `{N: 1, D: 20000001}` (showing the data loss is in the scalar field only). Optionally, also test with `ManageSellOffer` using `Price{1, 20000001}` to show the cross-operation scope.
