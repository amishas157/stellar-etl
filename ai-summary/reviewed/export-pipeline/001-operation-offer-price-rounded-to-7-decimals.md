# H001: Offer-operation `details.price` rounds non-zero prices down to `0`

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_operations` should preserve the effective offer price for `manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` rows. A non-zero on-chain price such as `1/2147483647` should export as a non-zero decimal, not collapse to `0`.

## Mechanism

`extractOperationDetails()` routes all three offer-creation/update operations through `addPriceDetails()`. That helper calls `price.String()` and then parses the result back into `float64`; upstream `xdr.Price.String()` formats the rational with `FloatString(7)`, so any price below `0.00000005` rounds to the literal string `0.0000000` before the ETL ever serializes it. The exported `details.price` field therefore becomes a believable but wrong zero for legitimate low-price offers.

## Trigger

Run `export_operations` over a ledger containing a `manage_sell_offer`, `manage_buy_offer`, or `create_passive_sell_offer` whose `Price` is a small but valid ratio such as `xdr.Price{N: 1, D: 2147483647}`. The resulting JSON/parquet `details.price` value should be non-zero, but the current code path will emit `0`.

## Target Code

- `internal/transform/operation.go:addPriceDetails:409-420` — converts `xdr.Price` to `details.price` via `price.String()` and `ParseFloat`
- `internal/transform/operation.go:extractOperationDetails:701-746` — routes `manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` through `addPriceDetails`

## Evidence

The production helper explicitly round-trips through `price.String()` instead of preserving the rational directly. `go doc -src github.com/stellar/go-stellar-sdk/xdr.Price.String` shows that `xdr.Price.String()` uses `big.NewRat(...).FloatString(7)`, so the truncation to seven decimal places is guaranteed before the ETL parses the value.

## Anti-Evidence

The same rows also carry `details.price_r`, which preserves the exact numerator and denominator. Downstream consumers that ignore `details.price` and reconstruct from `price_r` can recover the correct value, but the named `price` field itself is still silently wrong.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full code path from `extractOperationDetails()` through `addPriceDetails()` at `operation.go:409-420`. Confirmed that `xdr.Price.String()` in the upstream `go-stellar-sdk` (pinned at `v0.0.0-20251211085638-ba09a6a91775`) uses `big.NewRat(int64(p.N), int64(p.D)).FloatString(7)`, which truncates to exactly 7 decimal places. Verified with Go code that `FloatString(7)` for `1/2147483647` produces `"0.0000000"` and `strconv.ParseFloat` yields `0`. The boundary is at D=20000001 (for N=1): any price with N/D < ~5e-8 silently becomes zero.

### Code Paths Examined

- `internal/transform/operation.go:409-420` (`addPriceDetails`) — calls `price.String()`, parses result with `ParseFloat`, stores as `details["price"]`. No guard against sub-precision values.
- `internal/transform/operation.go:701-711` — `ManageBuyOffer` routes through `addPriceDetails`
- `internal/transform/operation.go:720-730` — `ManageSellOffer` routes through `addPriceDetails`
- `internal/transform/operation.go:739-748` — `CreatePassiveSellOffer` routes through `addPriceDetails`
- `internal/transform/operation.go:1008-1013` — `LiquidityPoolDeposit` `min_price` and `max_price` also route through `addPriceDetails` (same bug applies)
- `go-stellar-sdk/xdr/price.go:9` — `Price.String()` implementation confirms `FloatString(7)`
- `internal/transform/schema.go:178-181` — `Price` struct for `price_r` preserves exact int32 N/D (confirms anti-evidence)

### Findings

1. **Bug confirmed**: `FloatString(7)` truncation is real and verified in Go. For `Price{N:1, D:2147483647}`, the output is `"0.0000000"` → parsed as float64 `0`. The exact boundary is: any `N/D < 1/20000000` (≈ 5e-8) rounds to zero.

2. **Affected operations**: Five call sites use `addPriceDetails`:
   - `manage_buy_offer` (line 709)
   - `manage_sell_offer` (line 728)
   - `create_passive_sell_offer` (line 746)
   - `liquidity_pool_deposit` min_price (line 1008)
   - `liquidity_pool_deposit` max_price (line 1011)

3. **Data loss is partial**: The `price_r` field (N/D as integers) is always written alongside `price`, so downstream consumers CAN recover the exact value. However, the `price` float field itself is silently corrupted to zero — any query or system using `details.price` directly gets wrong data.

4. **Realistic trigger**: Stellar allows arbitrary `Int32` N and D values in `xdr.Price`. While extreme ratios like `1/2147483647` are uncommon on mainnet, custom tokens with very low relative value do exist, and the SDEX (Stellar Decentralized Exchange) has no protocol-level minimum price. Any price ratio below ~5e-8 triggers this bug.

5. **Severity assessment**: Critical is appropriate. This is a financial field (`price`) that silently produces a wrong numeric value (`0` instead of a non-zero price). The export looks valid — no error, no warning — but the data is incorrect. Downstream analytics or compliance systems using the `price` field directly would compute wrong values.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Create a `ManageSellOfferOp` with `Price: xdr.Price{N: 1, D: 2147483647}` wrapped in the standard `OperationTransformInput` fixture. Use an existing offer operation test as a template.
- **Steps**: Call `TransformOperation()` with the crafted input. Extract `details["price"]` from the result.
- **Assertion**: Assert that `details["price"].(float64) != 0` — currently this will FAIL because the price rounds to zero. Also verify `details["price_r"]` preserves `{N: 1, D: 2147483647}` correctly (this should pass, confirming the anti-evidence).
