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
