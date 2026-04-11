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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced from `TransformTrade` (trade.go:21) through the synthetic offer path (trade.go:118-119) into `EncodeOfferId` (synt_offer_id.go:28-34). Confirmed the technical analysis is correct: `EncodeOfferId` checks `id & 0xC000000000000000 != 0` and panics if bit 62 or 63 is set. A TOID for ledger 1073741824 (2^30) produces `0x4000000000000000`, which sets bit 62. However, this is an explicitly documented, intentional design constraint ported from Stellar Horizon.

### Code Paths Examined

- `internal/toid/synt_offer_id.go:28-34` — `EncodeOfferId` reserves bit 62 for the type flag and documents the resulting ceiling of ledger 1073741823 (~170 years)
- `internal/toid/synt_offer_id.go:12-27` — Comment explicitly documents the reduced ceiling and its 170-year timeline
- `internal/transform/trade.go:116-119` — `TransformTrade` calls `EncodeOfferId(uint64(operationID)+1, toid.TOIDType)` when `BuyingOffer == nil`
- `internal/toid/main.go:139-157` — `ToInt64()` packs ledger into bits 63-32, confirming ledger 2^30 sets bit 62

### Why It Failed

This is **working-as-designed behavior**, not a bug. Three factors make this NOT_VIABLE:

1. **Documented design constraint**: The comment at `synt_offer_id.go:17-27` explicitly documents the reduced ceiling (ledger 1073741823) and calculates it won't be reached for ~170 years. This is an intentional trade-off of reserving bit 62 for the type flag.

2. **No silent data corruption**: When triggered, the code panics (crashes) rather than producing wrong output. A panic is a safety mechanism, not a data correctness bug. The ETL's objective is correctness, and crashing is correct behavior when an assumption is violated.

3. **Upstream design**: The comment states this is "Taken from https://github.com/stellar/stellar-horizon/tree/master/internal/db2/history" — this is ported Horizon logic with an accepted network-wide constraint, not an ETL-specific oversight.

The hypothesis's "Expected Behavior" asserts that synthetic offers should work for the full TOID range, but that is not the design intent. The design intentionally sacrifices half the ledger range for synthetic offer IDs to gain a type-discriminated encoding.

### Lesson Learned

Documented capacity limits with explicit timelines (especially ~170 years) are design constraints, not bugs. A future hypothesis should focus on cases where the code silently produces wrong data, not where it deliberately crashes on a well-documented boundary.
