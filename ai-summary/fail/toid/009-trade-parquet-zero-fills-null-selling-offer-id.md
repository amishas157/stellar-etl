# H003: Liquidity-pool trade parquet rows fabricate `selling_offer_id = 0`

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a trade is produced by a liquidity-pool claim rather than a classic orderbook offer, parquet output should preserve `selling_offer_id = NULL` because there is no core offer ID or synthetic TOID-backed offer ID for the selling side of that trade.

## Mechanism

`TransformTrade()` leaves `SellingOfferID` unset for `ClaimAtomTypeLiquidityPool` rows and instead records the pool identifier fields. But `TradeOutputParquet` stores `selling_offer_id` as a plain `int64`, and `TradeOutput.ToParquet()` copies `to.SellingOfferID.Int64` without checking `to.SellingOfferID.Valid`. Every liquidity-pool trade therefore exports a fake selling offer ID of `0`, which looks like a normal integer key and can poison downstream joins or dedup logic.

## Trigger

Run `export_trades --write-parquet` on any ledger containing a liquidity-pool trade (`claimOffer.Type == xdr.ClaimAtomTypeClaimAtomTypeLiquidityPool`).

## Target Code

- `internal/transform/trade.go:80-114` — the liquidity-pool branch sets pool fields and intentionally leaves `outputSellingOfferID` unset.
- `internal/transform/schema.go:286-312` — JSON schema models `selling_offer_id` as `null.Int`.
- `internal/transform/schema_parquet.go:219-244` — parquet schema uses `SellingOfferID int64`.
- `internal/transform/parquet_converter.go:248-274` — parquet conversion writes `to.SellingOfferID.Int64` unconditionally.
- `cmd/export_trades.go:55-70` — trade rows are written through the parquet converter when `--write-parquet` is enabled.

## Evidence

In the non-liquidity-pool branch, `outputSellingOfferID` is set from `claimOffer.OfferId()`. In the liquidity-pool branch it is never assigned, so the `null.Int` remains invalid in JSON. The parquet conversion path discards that validity bit and emits the zero-value integer instead.

## Anti-Evidence

If downstream consumers already treat `selling_liquidity_pool_id` as authoritative for pool trades, they may be able to ignore `selling_offer_id`. But the parquet column still contains a wrong identifier rather than the intended null.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of fail/toid/008-contract-event-parquet-collapses-null-operationid-to-zero
**Failed At**: reviewer

### Trace Summary

Traced `TransformTrade()` at `trade.go:80-114` confirming the liquidity-pool branch leaves `outputSellingOfferID` as its zero-value `null.Int{Valid: false, Int64: 0}`. Then traced `TradeOutput.ToParquet()` at `parquet_converter.go:248-274` confirming it assigns `to.SellingOfferID.Int64` (yielding `0`). The mechanism described in the hypothesis is factually correct. However, fail finding 008 already investigated this exact code path and the broader null→0 parquet pattern, concluding it is a deliberate, codebase-wide design choice.

### Code Paths Examined

- `internal/transform/trade.go:80-114` — liquidity-pool branch leaves `outputSellingOfferID` unset (confirmed)
- `internal/transform/trade.go:110-111` — non-pool branch sets `outputSellingOfferID = null.IntFrom(int64(claimOffer.OfferId()))`
- `internal/transform/parquet_converter.go:266` — `SellingOfferID: to.SellingOfferID.Int64` unconditionally copies the inner int64 (confirmed)
- `internal/transform/schema_parquet.go:236` — `SellingOfferID int64` uses non-nullable type (confirmed)
- `internal/transform/schema.go:303` — `SellingOfferID null.Int` uses nullable type in JSON (confirmed)

### Why It Failed

Duplicate of fail/toid/008. That investigation explicitly examined `TradeOutput.ToParquet()` lines 266-272, listing `SellingOfferID`, `BuyingOfferID`, `LiquidityPoolFee`, and `RoundingSlippage` as instances of the null.Int→bare int64 pattern. It concluded that every `null.Int` in the codebase's JSON schema becomes a bare `int64` in the parquet schema — no parquet field anywhere uses `OPTIONAL` repetition type for nullable scalars. This is a deliberate, systematic design choice, not a per-field bug. Hypothesis 003 is a specific instance of the same pattern already deemed working-as-designed.

### Lesson Learned

When a codebase-wide pattern (null→zero-value in parquet) has already been investigated and classified as by-design, individual instances of that pattern in other converters are duplicates, not independent findings. Always check existing fail investigations for pattern-level conclusions before filing field-specific hypotheses.
