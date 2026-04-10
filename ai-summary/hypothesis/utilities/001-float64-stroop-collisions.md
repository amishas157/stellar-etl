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
