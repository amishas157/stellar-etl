# H002: Trade Parquet Collapses Nullable Routing Fields Into Zeros

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Trade exports should preserve whether a row came from an offer book or a liquidity pool. For offer-book trades, `selling_liquidity_pool_id`, `liquidity_pool_fee`, `rounding_slippage`, and `seller_is_exact` should stay absent. For liquidity-pool trades, `selling_offer_id` should stay absent because there is no seller offer to join against.

## Mechanism

`TradeOutput` models these fields with `null.String`, `null.Int`, and `null.Bool`, but `TradeOutputParquet` replaces them with plain `string`, `int64`, and `bool`. `TradeOutput.ToParquet()` then serializes `.String`, `.Int64`, and `.Bool`, so absent JSON values become `""`, `0`, and `false` in Parquet, silently rewriting row semantics and making offer trades look like pool trades with zero-valued attributes (or vice versa).

## Trigger

1. Run `export_trades` with `--parquet-output` on a ledger containing both offer-book trades and liquidity-pool trades.
2. Inspect an offer-book row: JSON omits `selling_liquidity_pool_id` / `liquidity_pool_fee`, but Parquet contains empty-string / zero defaults.
3. Inspect a liquidity-pool row: JSON omits `selling_offer_id`, but Parquet contains `0`.

## Target Code

- `internal/transform/trade.go:80-99` — liquidity-pool-only fields are populated conditionally
- `internal/transform/trade.go:110-157` — offer trades populate `selling_offer_id` while liquidity-pool trades leave it null
- `internal/transform/schema.go:303-311` — JSON schema marks these fields nullable
- `internal/transform/schema_parquet.go:236-243` — Parquet schema makes the same fields non-nullable primitives
- `internal/transform/parquet_converter.go:248-274` — converter flattens nulls to primitive zero values
- `cmd/export_trades.go:68-70` — live Parquet export path for trades

## Evidence

The transformer clearly uses nullability to distinguish trade shapes: liquidity-pool trades set `SellingLiquidityPoolID`, `LiquidityPoolFee`, `RoundingSlippage`, and `SellerIsExact`, while offer trades set `SellingOfferID`. The Parquet schema removes that nullability entirely, and the converter extracts raw primitive zero values from the null wrappers, so downstream Parquet readers lose the original routing information even though JSON preserved it.

## Anti-Evidence

Some zero values are valid in-domain for rows where the field is present, such as `rounding_slippage = 0` on a liquidity-pool trade. The bug is therefore not "zero is always wrong"; it is that Parquet cannot distinguish "field absent" from "field present with zero/false/empty" on the affected trade shapes.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `TransformTrade()` in `trade.go` through the `TradeOutput` JSON schema in `schema.go`, the `TradeOutputParquet` Parquet schema in `schema_parquet.go`, and the `ToParquet()` converter in `parquet_converter.go`. Confirmed that six nullable fields (`SellingOfferID`, `BuyingOfferID`, `SellingLiquidityPoolID`, `LiquidityPoolFee`, `RoundingSlippage`, `SellerIsExact`) are modeled as `null.Int`/`null.String`/`null.Bool` in the JSON schema but as plain `int64`/`string`/`bool` in the Parquet schema. The converter extracts `.Int64`, `.String`, and `.Bool` directly from the null wrappers, yielding zero values (`0`, `""`, `false`) when `Valid == false`. The `guregu/null` library embeds `sql.NullInt64`/`sql.NullBool` whose zero-value inner fields are `0`/`false`.

### Code Paths Examined

- `internal/transform/trade.go:80-114` — LP branch sets `liquidityPoolID`, `outputPoolFee`, `roundingSlippageBips` via `null.IntFrom()`/`null.StringFrom()`; offer branch sets `outputSellingOfferID` via `null.IntFrom()`. Unset fields remain as zero-value `null.X{}` (Valid=false).
- `internal/transform/trade.go:164-253` — `extractClaimedOffers()` sets `sellerIsExact` only for PathPaymentStrictSend (false) and PathPaymentStrictReceive (true); for ManageBuyOffer/ManageSellOffer/CreatePassiveSellOffer it stays as zero-value `null.Bool{}`.
- `internal/transform/schema.go:303-311` — `TradeOutput` declares `SellingOfferID null.Int`, `SellingLiquidityPoolID null.String`, `LiquidityPoolFee null.Int`, `RoundingSlippage null.Int`, `SellerIsExact null.Bool`.
- `internal/transform/schema_parquet.go:236-243` — `TradeOutputParquet` declares the same fields as plain `int64`, `string`, `int64`, `int64`, `bool`.
- `internal/transform/parquet_converter.go:248-274` — `ToParquet()` extracts `.Int64`, `.String`, `.Bool` directly, collapsing null to zero values.
- `github.com/guregu/null@v4.0.0/bool.go:14-16` and `int.go:15-17` — Confirmed `null.Bool` wraps `sql.NullBool{Bool: false, Valid: false}` and `null.Int` wraps `sql.NullInt64{Int64: 0, Valid: false}`.

### Findings

**Six fields lose null semantics in Parquet output:**

| Field | JSON (offer trade) | Parquet (offer trade) | JSON (LP trade) | Parquet (LP trade) |
|-------|--------------------|-----------------------|-----------------|--------------------|
| `selling_offer_id` | present (valid ID) | valid ID | null (absent) | `0` |
| `selling_liquidity_pool_id` | null (absent) | `""` | present (pool hex) | pool hex |
| `liquidity_pool_fee` | null (absent) | `0` | present (e.g. 30) | 30 |
| `rounding_slippage` | null (absent) | `0` | present (incl. valid 0) | value (incl. valid 0) |
| `seller_is_exact` | null (absent) for manage ops | `false` | null or valid | value or `false` |
| `buying_offer_id` | always set | always set | always set | always set |

**Most impactful collision — `seller_is_exact`:** ManageBuyOffer and ManageSellOffer trades get `seller_is_exact = false` in Parquet (collapsed from null), which is indistinguishable from PathPaymentStrictSend's legitimate `false`. The `trade_type` field (1=offer, 2=LP) does NOT disambiguate here because all three operation types produce trade_type=1 offer trades.

**`rounding_slippage` ambiguity:** LP trades with genuine `rounding_slippage = 0` are indistinguishable from offer trades where the field should be absent. Downstream queries like "find LP trades with zero rounding slippage" would incorrectly include all offer trades.

While `trade_type` partially mitigates some cases (you can filter by trade_type to know LP fields are irrelevant for type=1), it cannot distinguish null from zero within the same trade type, and `seller_is_exact` crosses trade types within type=1.

### PoC Guidance

- **Test file**: `internal/transform/trade_test.go` (append new test case)
- **Setup**: Create two `TradeOutput` values — one for an offer-book trade (LP fields null) and one for an LP trade (selling_offer_id null, seller_is_exact null). Call `ToParquet()` on both.
- **Steps**: (1) Construct a `TradeOutput` with `SellingLiquidityPoolID: null.String{}` (not set) and `SellingOfferID: null.IntFrom(12345)`. Call `ToParquet()`. (2) Construct a `TradeOutput` with `SellingLiquidityPoolID: null.StringFrom("abc")` and `SellingOfferID: null.Int{}` (not set). Call `ToParquet()`. (3) Construct a `TradeOutput` with `SellerIsExact: null.Bool{}` (ManageBuyOffer — should be absent). Call `ToParquet()`.
- **Assertion**: Assert that the offer-trade Parquet has `SellingLiquidityPoolID == ""` and `LiquidityPoolFee == 0` (demonstrating collapse). Assert that the LP-trade Parquet has `SellingOfferID == 0` (demonstrating collapse). Assert that the ManageBuyOffer Parquet has `SellerIsExact == false` — same as PathPaymentStrictSend's `null.BoolFrom(false)` after conversion, proving semantic collision.
