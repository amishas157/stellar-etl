# H001: Float64 stroop conversion collapses distinct on-chain amounts

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When the ETL exports an on-chain `Int64` stroop amount, the output should preserve the full 7-decimal Stellar value so that two source amounts that differ by 1 stroop remain distinguishable downstream. For example, `922337203685477580` and `922337203685477581` stroops should export as distinct values, not the same JSON/Parquet number.

## Mechanism

`ConvertStroopValueToReal` converts `xdr.Int64` into `float64` via `big.Rat.Float64()`. IEEE-754 doubles cannot preserve single-stroop precision for large 64-bit amounts, so distinct source values collapse to the same exported number. Because this helper feeds balances, liabilities, reserves, and trade amounts, the ETL can silently publish wrong financial values that still look plausible.

## Trigger

Export any ledger containing a large issued-asset balance, trustline balance, liquidity-pool reserve, or trade amount near the upper `Int64` range. A concrete reproducer is two ledger entries whose source amounts are `922337203685477580` and `922337203685477581` stroops: both serialize to the same `float64` (`92233720368.54776`) after `ConvertStroopValueToReal`.

## Target Code

- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — converts exact stroop integers into `float64`
- `internal/transform/account.go:TransformAccount:86-90` — exports account balances and liabilities with the helper
- `internal/transform/trustline.go:TransformTrustline:70-79` — exports trustline balances and liabilities with the helper
- `internal/transform/trade.go:TransformTrade:132-145` — exports trade amounts with the helper

## Evidence

`ConvertStroopValueToReal` returns `float64` with no exactness check. I verified that `922337203685477580`, `922337203685477581`, `922337203685477582`, and `922337203685477600` stroops all produce the same exported `float64` representation (`92233720368.54776`). The output schemas for accounts, trustlines, pools, offers, trades, and many operation details all consume this helper directly.

## Anti-Evidence

For smaller amounts, `float64` preserves enough precision that the output appears correct. The codebase also intentionally models these fields as decimal-looking numbers rather than strings, so some downstream consumers may tolerate approximate values — but that does not prevent silent corruption for high-magnitude amounts.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced from `ConvertStroopValueToReal` (internal/utils/main.go:85-88) through all 30+ callers across 6 transform files. The function converts `xdr.Int64` to `float64` via `big.NewRat(int64(input), int64(10000000)).Float64()`, discarding the `Accuracy` return value that would indicate precision loss. The resulting `float64` feeds every financial amount field in the output schema — Balance, BuyingLiabilities, SellingLiabilities, Amount, SellingAmount, BuyingAmount — and no parallel integer/string representation exists in any output struct. I confirmed via Go execution that the first single-stroop collision occurs at 10,735,946,655,003,267 stroops (~1.07 billion real units), with progressively worse precision loss at higher magnitudes.

### Code Paths Examined

- `internal/utils/main.go:ConvertStroopValueToReal:85-88` — `big.Rat.Float64()` exactness flag discarded with `_`; returns lossy `float64`
- `internal/transform/schema.go:99-101` — `AccountOutput.Balance`, `BuyingLiabilities`, `SellingLiabilities` all `float64` with no parallel integer field
- `internal/transform/schema.go:245,248-249` — `TrustlineOutput.Balance`, `BuyingLiabilities`, `SellingLiabilities` all `float64`
- `internal/transform/schema.go:272` — `OfferOutput.Amount` is `float64`
- `internal/transform/schema.go:294,300` — `TradeOutput.SellingAmount`, `BuyingAmount` both `float64`
- `internal/transform/schema_parquet.go:79-81,179,182-183,205,227,233` — Parquet schema mirrors JSON: all amount fields are `type=DOUBLE`
- `internal/transform/account.go:88-90` — `TransformAccount` passes balance and liabilities through the lossy converter
- `internal/transform/trustline.go:75,78-79` — `TransformTrustline` passes balance and liabilities through the lossy converter
- `internal/transform/liquidity_pool.go:71,76,81` — Pool shares and reserves through lossy converter
- `internal/transform/trade.go:139,145` — Trade selling/buying amounts through lossy converter
- `internal/transform/offer.go:90` — Offer amount through lossy converter
- `internal/transform/operation.go:600-1061` — 20+ operation detail fields (starting_balance, amount, source_amount, limit, reserve amounts, shares) through lossy converter

### Findings

**Confirmed precision loss with exact collision threshold:**

| Stroop Value | Real Value (XLM) | Collides with next stroop? |
|---|---|---|
| 10,000,000,000,000 | 1,000,000 | No |
| 100,000,000,000,000 | 10,000,000 | No |
| 1,000,000,000,000,000 | 100,000,000 | No |
| 10,735,946,655,003,267 | ~1,073,594,665 | **YES — first collision** |
| 50,000,000,000,000,000 | 5,000,000,000 | YES |
| 922,337,203,685,477,580 | ~92,233,720,369 | YES |

The collision threshold of ~1.07 billion real units is reachable for:
- **XLM accounts**: Total supply is ~50B XLM. Major custodians and exchanges hold billions.
- **Custom asset trustlines**: Issuers can set arbitrary supplies; stablecoins and wrapped tokens commonly exceed 1B units.
- **Liquidity pool reserves**: Large pools can hold reserves above the threshold.

**No alternate representation**: The output schemas (JSON and Parquet) provide ONLY `float64` for these fields. There is no `balance_stroops` integer field. Downstream consumers have no way to recover the exact source value.

**Discarded exactness flag**: The `big.Rat.Float64()` method returns `(float64, Accuracy)` where `Accuracy` indicates if the conversion was exact. The code discards this with `output, _ :=`, eliminating the only mechanism that could detect and report precision loss.

**Upstream precedent**: The Stellar Horizon API returns amounts as strings (e.g., `"100.0000000"`) specifically to avoid IEEE-754 precision loss. This ETL's choice of `float64` provides strictly less precision than the standard Stellar API.

### PoC Guidance

- **Test file**: `internal/utils/main_test.go` (append to existing test file)
- **Setup**: No special setup needed. Import `math/big` and `github.com/stellar/go/xdr`.
- **Steps**:
  1. Call `ConvertStroopValueToReal(xdr.Int64(10735946655003267))` — the first colliding value
  2. Call `ConvertStroopValueToReal(xdr.Int64(10735946655003268))` — the next stroop value
  3. Also test extreme case: `ConvertStroopValueToReal(xdr.Int64(922337203685477580))` and `+1`
- **Assertion**: Assert that both calls return the same `float64` value (proving collision). Assert `result1 == result2` where the source stroops differ by 1. This demonstrates that the ETL produces identical exported values for distinct on-chain amounts, confirming silent financial data corruption.
