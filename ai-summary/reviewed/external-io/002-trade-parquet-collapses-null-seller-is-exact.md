# H002: Trade Parquet collapses nullable `seller_is_exact` to `false`

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_trades.seller_is_exact` should remain tri-state across export formats: `true` for strict-receive path payments, `false` for strict-send path payments, and `null` for non-path-payment trades such as manage-offer and passive-offer crossings. Parquet rows should preserve that same distinction instead of inventing a boolean value for rows that were null in JSON.

## Mechanism

`extractClaimedOffers()` only sets `sellerIsExact` on the two path-payment result arms, leaving ordinary orderbook trades with `null.Bool{Valid:false}`. `TradeOutput.ToParquet()` then writes `to.SellerIsExact.Bool` into a plain `bool` field in `TradeOutputParquet`, so null becomes Go's zero value `false`. That makes non-path-payment trades indistinguishable from strict-send path-payment trades in Parquet, even though the JSON export correctly keeps them null.

## Trigger

Run `export_trades --write-parquet` on any ledger containing a successful `manage_sell_offer`, `manage_buy_offer`, or `create_passive_sell_offer` trade. The JSON row should have `"seller_is_exact": null`, but the Parquet row should serialize `seller_is_exact = false`.

## Target Code

- `internal/transform/trade.go:164-250` - only path-payment branches populate `sellerIsExact`; all other trade-producing operations leave it null
- `internal/transform/schema.go:303-310` - JSON trade schema defines `SellerIsExact` as `null.Bool`
- `internal/transform/schema_parquet.go:240-243` - Parquet schema stores `seller_is_exact` as a plain `bool`
- `internal/transform/parquet_converter.go:266-274` - converter writes `to.SellerIsExact.Bool`, collapsing null to `false`
- `cmd/export_trades.go:61-70` - command emits the lossy Parquet rows when `--write-parquet` is enabled

## Evidence

The transform layer explicitly models `seller_is_exact` as nullable, and only path-payment operations assign a real boolean. The Parquet layer drops that nullability by using a non-nullable `bool` and reading the zero-valued `.Bool` field directly.

## Anti-Evidence

Rows produced by path payments still preserve `true` and `false` correctly. The corruption is specific to non-path-payment trades whose correct value is null in JSON.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete data flow from `extractClaimedOffers()` through `TransformTrade()` to `TradeOutput.ToParquet()`. Confirmed that the `sellerIsExact` return value is only set for `PathPaymentStrictSend` (line 227, `null.BoolFrom(false)`) and `PathPaymentStrictReceive` (line 243, `null.BoolFrom(true)`). For all three offer-type operations (`ManageBuyOffer`, `ManageSellOffer`, `CreatePassiveSellOffer`), the named return `sellerIsExact` retains its zero value `null.Bool{Bool: false, Valid: false}`. The Parquet converter at line 273 reads `.Bool` directly from this zero-valued struct, producing `false` instead of preserving the null semantics.

### Code Paths Examined

- `internal/transform/trade.go:164-262` — `extractClaimedOffers()`: confirmed only `PathPaymentStrictSend` (line 227) and `PathPaymentStrictReceive` (line 243) set `sellerIsExact`; `ManageBuyOffer` (line 181-193), `ManageSellOffer` (line 195-208), and `CreatePassiveSellOffer` (line 210-223) all return without setting it
- `internal/transform/trade.go:34` — caller receives `sellerIsExact` and passes it directly to `TradeOutput` at line 155
- `internal/transform/schema.go:310` — `SellerIsExact null.Bool` field in `TradeOutput` — nullable, correctly serializes to JSON `null` when `Valid == false`
- `internal/transform/schema_parquet.go:243` — `SellerIsExact bool` in `TradeOutputParquet` — non-nullable `bool`, cannot represent null
- `internal/transform/parquet_converter.go:273` — `SellerIsExact: to.SellerIsExact.Bool` — reads the `.Bool` field of a `null.Bool` where `Valid == false`, yielding Go zero value `false`

### Findings

The bug is confirmed. The `null.Bool` → `bool` conversion at the Parquet layer silently collapses null to `false`:

1. **JSON path (correct)**: `null.Bool{Valid: false}` marshals to JSON `null` via the `guregu/null` package's `MarshalJSON` method.
2. **Parquet path (buggy)**: `to.SellerIsExact.Bool` extracts the inner `bool` value, which is `false` when `Valid == false`. The Parquet schema uses a plain `BOOLEAN` column with no notion of nullability, so `false` is written.
3. **Data corruption**: In the Parquet output, non-path-payment trades (manage buy/sell offer, create passive sell offer) are indistinguishable from strict-send path payments. Both show `seller_is_exact = false`. This is a structural data correctness issue for any downstream consumer (e.g., BigQuery analytics) that uses Parquet exports to distinguish trade types by this field.

This is consistent with the known pattern where JSON and Parquet schemas intentionally diverge on some fields, but in this case the divergence causes semantic data loss rather than a deliberate omission of a convenience field.

### PoC Guidance

- **Test file**: `internal/transform/trade_test.go`
- **Setup**: Create a mock `ManageSellOffer` trade transaction (which produces `seller_is_exact = null` in JSON). Use existing test helpers to construct an `ingest.LedgerTransaction` with a successful manage-sell-offer result that claims at least one offer.
- **Steps**: (1) Call `TransformTrade()` with the mock transaction. (2) Verify the returned `TradeOutput` has `SellerIsExact.Valid == false`. (3) Call `.ToParquet()` on that output. (4) Cast the result to `TradeOutputParquet`.
- **Assertion**: Assert that `TradeOutputParquet.SellerIsExact == false` while `TradeOutput.SellerIsExact.Valid == false`, demonstrating that a null value was collapsed to `false` in the Parquet representation. This proves the Parquet output cannot distinguish null from false.
