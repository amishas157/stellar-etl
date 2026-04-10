# H001: Trade Parquet flattens nullable `seller_is_exact` to `false`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_trades.seller_is_exact` should preserve the three states already modeled by the JSON transform: `true` for strict-receive path payments, `false` for strict-send path payments, and `null` when the concept does not apply to the trade row at all. Manage-offer trades should stay distinguishable from strict-send path-payment trades.

## Mechanism

`extractClaimedOffers()` only sets `sellerIsExact` for the two path-payment operation types and leaves it unset for manage-offer trades. `TradeOutput` preserves that nullability with `null.Bool`, but `TradeOutputParquet` narrows the field to a required `bool` and `TradeOutput.ToParquet()` writes `to.SellerIsExact.Bool`, so every null JSON value becomes `false` in Parquet.

## Trigger

Export trades with `--write-parquet` for a ledger containing both a manage-offer trade and a path-payment-strict-send trade. The JSON rows will differ (`seller_is_exact=null` versus `seller_is_exact=false`), but the Parquet rows will serialize both as `false`.

## Target Code

- `internal/transform/trade.go:131-156` — writes `SellerIsExact` into `TradeOutput`
- `internal/transform/trade.go:225-244` — only path-payment code paths populate `sellerIsExact`
- `internal/transform/schema.go:303-311` — JSON schema models `seller_is_exact` as `null.Bool`
- `internal/transform/schema_parquet.go:236-243` — Parquet schema makes `seller_is_exact` a required `bool`
- `internal/transform/parquet_converter.go:266-273` — converts `SellerIsExact` with `to.SellerIsExact.Bool`
- `internal/transform/trade_test.go:721-764` — manage-offer fixture leaves `SellerIsExact` unset
- `internal/transform/trade_test.go:818-834` — path-payment fixtures set `SellerIsExact` to explicit `false` and `true`

## Evidence

The trade fixtures already show both source states: plain manage-offer rows omit `SellerIsExact`, while path-payment rows explicitly set it to `null.BoolFrom(false)` or `null.BoolFrom(true)`. The Parquet layer cannot represent that tri-state because the schema uses a bare `bool` and the converter reads the zero-value `false` from invalid `null.Bool` values.

## Anti-Evidence

Downstream consumers can still recover some context from `history_operation_id` or the originating operation type. But the exporter already chose to expose `seller_is_exact` as its own nullable field in JSON, so collapsing null into `false` in Parquet still turns a non-applicable attribute into a misleading negative value.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete data flow from `extractClaimedOffers()` through `TradeOutput` construction and `ToParquet()` conversion. Confirmed that `sellerIsExact` is a named return in `extractClaimedOffers()` that stays at its zero value `null.Bool{}` (Valid=false, Bool=false) for all manage-offer operation types (ManageBuyOffer, ManageSellOffer, CreatePassiveSellOffer), is set to `null.BoolFrom(false)` for PathPaymentStrictSend, and `null.BoolFrom(true)` for PathPaymentStrictReceive. The Parquet schema uses a required `bool` and the converter reads `.Bool` directly, collapsing the null and explicit-false states into the same Parquet value `false`.

### Code Paths Examined

- `internal/transform/trade.go:164-262` — `extractClaimedOffers()` uses named return `sellerIsExact null.Bool`; manage-offer cases (lines 181-223) return without setting it; strict-send (line 227) sets `null.BoolFrom(false)`; strict-receive (line 243) sets `null.BoolFrom(true)`
- `internal/transform/trade.go:131-157` — `TransformTrade()` assigns `SellerIsExact: sellerIsExact` directly from the function return
- `internal/transform/schema.go:310` — `TradeOutput.SellerIsExact` is `null.Bool`, correctly modeling the tri-state
- `internal/transform/schema_parquet.go:243` — `TradeOutputParquet.SellerIsExact` is `bool` (required, non-nullable)
- `internal/transform/parquet_converter.go:273` — `SellerIsExact: to.SellerIsExact.Bool` reads the inner bool field, ignoring `.Valid`

### Findings

The bug follows the exact same pattern as the confirmed success/003 finding (transaction `min_account_sequence` nullable collapse). For trade rows:

| Operation Type | JSON `seller_is_exact` | Parquet `seller_is_exact` |
|---|---|---|
| ManageBuyOffer | `null` | `false` |
| ManageSellOffer | `null` | `false` |
| CreatePassiveSellOffer | `null` | `false` |
| PathPaymentStrictSend | `false` | `false` |
| PathPaymentStrictReceive | `true` | `true` |

Manage-offer trades (null) and strict-send path payments (false) become indistinguishable in Parquet output. This is structural data corruption — a "not applicable" value is silently converted to a "no" value.

### PoC Guidance

- **Test file**: `internal/transform/trade_test.go` (or a new `internal/transform/data_integrity_poc_test.go`)
- **Setup**: Use the existing `inputEnvelope` test helper with a ManageBuyOffer operation to produce a TradeOutput where `SellerIsExact` is `null.Bool{}`, and a PathPaymentStrictSend operation to produce a TradeOutput where `SellerIsExact` is `null.BoolFrom(false)`
- **Steps**: Call `TransformTrade()` for both operation types, then call `.ToParquet()` on each result
- **Assertion**: Assert that the JSON-layer outputs differ (`SellerIsExact.Valid` is false vs true), then assert that the Parquet outputs also differ — the test should fail, demonstrating the collapse: `parquetManageOffer.SellerIsExact == parquetStrictSend.SellerIsExact` both equal `false`
