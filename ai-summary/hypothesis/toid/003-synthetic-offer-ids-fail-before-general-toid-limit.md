# H003: Synthetic trade offer IDs fail roughly halfway to the normal TOID ceiling

**Date**: 2026-04-10
**Subsystem**: toid
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Trade export should keep producing a valid `buying_offer_id` for immediately filled offers for every ledger where the underlying `history_operation_id` is still representable. Downstream analytics should not lose only the synthetic-offer subset while the rest of the TOID-based export surface still works.

## Mechanism

`EncodeOfferId()` reserves bit 62 as the synthetic-offer type flag and panics whenever the underlying TOID already has either of the top two bits set. `TransformTrade()` derives synthetic offer IDs from `uint64(operationID)+1`, so once ledger sequence reaches `1073741824`, operation TOIDs begin setting bit 62 and synthetic offer generation panics, even though normal ledger/transaction/operation TOIDs remain usable until `2147483647`. The trade pipeline therefore loses immediately-filled offers substantially earlier than other TOID-backed exports.

## Trigger

Process a successful trade-producing operation in ledger `1073741824` or later where `BuyingOffer == nil`, forcing `TransformTrade()` down the synthetic offer ID path.

## Target Code

- `internal/toid/synt_offer_id.go:14-33` — synthetic offer encoding reserves bit 62 and panics when the input TOID uses that space.
- `internal/transform/trade.go:116-120` — trade export synthesizes `BuyingOfferID` from the operation TOID.

## Evidence

The comment in `synt_offer_id.go` documents the reduced ceiling (`1073741823`) but the trade exporter does not gate, degrade, or surface that limit; it simply calls `EncodeOfferId()` during normal row construction. This creates a real divergence where standard TOID fields remain encodable but synthetic trade IDs stop being exportable.

## Anti-Evidence

The limit is documented in the helper comment, so a reviewer may decide this is an accepted design constraint rather than an accidental bug. The missing piece to validate is whether the ETL is expected to preserve synthetic-offer behavior for the full normal TOID range.
