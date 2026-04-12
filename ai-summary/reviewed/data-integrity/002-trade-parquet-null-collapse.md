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

Traced the complete path from `TransformTrade()` through the conditional field population in `trade.go:80-114`, through `TradeOutput` schema definition using `null.Int/String/Bool` in `schema.go:303-311`, into the Parquet conversion at `parquet_converter.go:248-274` where `.Int64`, `.String`, and `.Bool` are accessed without checking `.Valid`. Confirmed that `guregu/null` types wrap `sql.NullInt64/NullString/NullBool` where the raw value field is Go zero when `.Valid` is false. The `xitongsys/parquet-go` library supports `repetitiontype=OPTIONAL` for nullable columns, but no trade Parquet fields use it.

### Code Paths Examined

- `internal/transform/trade.go:80-114` — LP trades set `liquidityPoolID`, `outputPoolFee`, `roundingSlippageBips` but leave `outputSellingOfferID` as zero-value `null.Int`; offer trades set `outputSellingOfferID` but leave LP fields as zero-value nulls
- `internal/transform/trade.go:164-259` — `extractClaimedOffers` only sets `sellerIsExact` for `PathPaymentStrictSend` (false) and `PathPaymentStrictReceive` (true); for `ManageBuyOffer`, `ManageSellOffer`, `CreatePassiveSellOffer` it stays as zero-value `null.Bool{}`
- `internal/transform/schema.go:303-311` — `SellingOfferID null.Int`, `BuyingOfferID null.Int`, `SellingLiquidityPoolID null.String`, `LiquidityPoolFee null.Int`, `RoundingSlippage null.Int`, `SellerIsExact null.Bool`
- `internal/transform/schema_parquet.go:236-243` — Same fields as plain `int64`, `string`, `bool` — no OPTIONAL repetition type
- `internal/transform/parquet_converter.go:266-273` — `to.SellingOfferID.Int64`, `to.SellingLiquidityPoolID.String`, `to.LiquidityPoolFee.Int64`, `to.RoundingSlippage.Int64`, `to.SellerIsExact.Bool` — accesses raw value fields, ignoring `.Valid`
- `guregu/null@v4.0.0/int.go` — `null.Int` wraps `sql.NullInt64{Int64: 0, Valid: false}`; accessing `.Int64` yields `0` regardless of `.Valid`

### Findings

1. **Six fields affected**: `SellingOfferID`, `BuyingOfferID`, `SellingLiquidityPoolID`, `LiquidityPoolFee`, `RoundingSlippage`, `SellerIsExact` all collapse null→zero in Parquet. (`BuyingOfferID` is always set via lines 116-120, so it's not practically affected.)

2. **`SellerIsExact` is the worst case**: For manage-offer operations (`ManageBuyOffer`, `ManageSellOffer`, `CreatePassiveSellOffer`), `sellerIsExact` is never assigned and stays `null.Bool{}`. In JSON this is omitted (null). In Parquet it becomes `false`, which is **indistinguishable** from `PathPaymentStrictSend` trades where `SellerIsExact` is genuinely `false`. A downstream consumer cannot tell whether `seller_is_exact = false` means "this is a strict-send path payment" or "this field does not apply."

3. **`TradeType` partially mitigates but doesn't eliminate the issue**: `TradeType` (1=offer, 2=LP) is always populated and can disambiguate offer vs LP routing. However, `SellerIsExact` cuts across both categories (it distinguishes strict-send vs strict-receive path payments, both of which can produce either offer or LP trades), so `TradeType` alone cannot disambiguate the `SellerIsExact` null-collapse.

4. **Not covered by prior investigation 010**: The prior transaction-precondition null-collapse (fail/data-integrity/010) was rejected because null and 0 are semantically equivalent for age/gap constraints. Here, null and zero/false are NOT equivalent — null means "field does not apply to this trade type" while zero/false carries specific meaning.

### PoC Guidance

- **Test file**: `internal/transform/trade_test.go`
- **Setup**: Create two `ingest.LedgerTransaction` objects: one with a `ManageSellOffer` operation that produces an offer-book trade (ClaimAtomTypeClaimAtomTypeOrderBook), and one with a path payment that produces an LP trade (ClaimAtomTypeClaimAtomTypeLiquidityPool). Use existing test helpers in `internal/utils/` for mock transaction construction.
- **Steps**:
  1. Call `TransformTrade()` for the offer-book transaction
  2. Call `.ToParquet()` on the resulting `TradeOutput`
  3. Assert that the Parquet `SellingLiquidityPoolID` is `""` and `LiquidityPoolFee` is `0` — demonstrating that null collapsed to zero values
  4. Call `TransformTrade()` for the LP transaction
  5. Call `.ToParquet()` on the resulting `TradeOutput`
  6. Assert that `SellingOfferID` is `0` — demonstrating null collapsed
  7. For a `ManageSellOffer` trade, verify that the JSON `SellerIsExact` is null (`.Valid == false`) but the Parquet value is `false` — identical to a genuine `PathPaymentStrictSend` trade's `SellerIsExact`
- **Assertion**: The core assertion is that `TradeOutput` fields with `.Valid == false` produce non-null zero values in the Parquet output, proving information loss. The strongest sub-assertion is that a ManageSellOffer trade and a PathPaymentStrictSend trade produce identical `seller_is_exact = false` in Parquet despite having different JSON representations (null vs false).
