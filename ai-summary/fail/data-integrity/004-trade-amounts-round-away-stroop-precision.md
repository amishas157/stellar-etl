# H004: Trade export rounds large claimed-offer amounts to different prices and amounts

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformTrade()` should preserve the exact claimed-offer amounts from the XDR `ClaimAtom`, so a sold or bought amount of `99_999_999_999_999_999` stroops should export as `9999999999.9999999`. The derived trade row should not round that amount up to `10000000000`, because downstream price and volume analytics rely on exact trade quantities.

## Mechanism

The trade transformer converts both `AmountSold()` and `AmountBought()` through `ConvertStroopValueToReal()`, then stores them in `float64` fields that are copied unchanged into Parquet. For sufficiently large trades, the 1-stroop distinction disappears in binary float, so the exported `selling_amount` / `buying_amount` no longer match the executed trade on-chain and any downstream price reconstruction from those rounded amounts becomes wrong.

## Trigger

1. Process a successful offer-claiming operation whose `ClaimAtom.AmountSold()` or `AmountBought()` is a large legal stroop value such as `99_999_999_999_999_999`.
2. Run `export_trades`.
3. Compare the exported trade row to the underlying claim atom: the amount field rounds to `10000000000` instead of preserving the exact `...9999999` decimal.

## Target Code

- `internal/utils/main.go:84-87` — exact stroops are narrowed to `float64`
- `internal/transform/trade.go:131-157` — trade rows assign `SellingAmount` and `BuyingAmount` from the lossy helper
- `internal/transform/schema.go:286-312` — trade amounts are stored as `float64`
- `internal/transform/parquet_converter.go:248-274` — Parquet conversion propagates the rounded amounts unchanged

## Evidence

The trade path is one of the few monetary exports where the table itself is used for direct volume analytics, but it keeps only floating-point quantities. A concrete Go run of the repository helper shows `99_999_999_999_999_999` stroops serializing as `10000000000`, so two distinct trade sizes become indistinguishable once exported. The previously confirmed trade Parquet null-collapse bug shows this table already has format-level integrity issues, making another field-level corruption path here plausible.

## Anti-Evidence

The exact rational price components `price_n` and `price_d` are still preserved, which partially mitigates price reconstruction for some workflows. Small and medium trade amounts also remain exact, so the corruption only appears on higher-value claim atoms and can evade existing low-range fixtures.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `success/utilities/001-float64-stroop-collisions.md.gh-published`
**Failed At**: reviewer

### Trace Summary

Traced `ConvertStroopValueToReal` in `internal/utils/main.go:84-87` through `TransformTrade` in `internal/transform/trade.go:131-145`, confirming the float64 precision loss mechanism is real. However, this exact mechanism — including trade amounts as explicitly listed affected code (`internal/transform/trade.go:131-145`) — was already confirmed, published, and documented as `success/utilities/001-float64-stroop-collisions.md.gh-published`.

### Code Paths Examined

- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal` uses `big.NewRat(int64(input), 10000000).Float64()` with exactness discarded
- `internal/transform/trade.go:52,64` — `outputSellingAmount` and `outputBuyingAmount` obtained from `ClaimAtom.AmountSold()` / `AmountBought()`
- `internal/transform/trade.go:139,145` — `SellingAmount` and `BuyingAmount` assigned via `utils.ConvertStroopValueToReal()`, producing lossy float64
- `internal/transform/schema.go:294,300` — `TradeOutput` stores both as `float64`
- `internal/transform/schema_parquet.go:227,233` — `TradeOutputParquet` stores both as `DOUBLE`
- `internal/transform/parquet_converter.go:257,263` — Parquet converter copies the already-rounded float64 values unchanged

### Why It Failed

This is an exact duplicate of the already confirmed and published finding `success/utilities/001-float64-stroop-collisions.md.gh-published`, which:
1. Identifies the same root cause: `ConvertStroopValueToReal` in `internal/utils/main.go:84-87`
2. Explicitly lists `internal/transform/trade.go:131-145` — "exports trade amounts through the helper" — as affected code
3. Covers the same `float64` / `DOUBLE` schema fields in `schema.go:237-300` and `schema_parquet.go:171-233`
4. Has been published with Critical severity

The prior account-balances variant (`fail/data-integrity/003`) was also rejected as a duplicate of this same cross-cutting finding. The root cause is shared across all `ConvertStroopValueToReal` callers and was comprehensively documented in the utilities subsystem success finding.

### Lesson Learned

Before submitting entity-specific variants of a precision-loss hypothesis, check `success/utilities/` for cross-cutting findings that already cover the shared helper function. The `ConvertStroopValueToReal` float64 limitation has been confirmed, published, and documented as covering all callers including accounts, trustlines, offers, and trades.
