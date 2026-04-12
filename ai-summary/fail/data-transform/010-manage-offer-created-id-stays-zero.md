# H002: Successful `manage_*_offer` create rows can export `offer_id = 0`

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a successful `manage_buy_offer` or `manage_sell_offer` operation creates a new offer, `history_operations.details.offer_id` should contain the newly created on-ledger offer ID from the operation result. The sentinel request value `OfferId = 0` means "create a new offer", so the exported success row should not keep `0` after the result returns the actual `OfferEntry.OfferId`.

## Mechanism

`extractOperationDetails()` always serializes `details["offer_id"]` from the request body (`op.OfferId`) and never consults the success result. But the XDR success union carries an `OfferEntry` for created and updated offers. On create-new requests, the input sentinel stays `0`, so successful operation rows can export `offer_id = 0` even though the result already contains the real created offer ID.

## Trigger

Submit a successful `manage_buy_offer` or `manage_sell_offer` with request `OfferId = 0` that creates a new offer (`ManageOfferEffectManageOfferCreated`). The current row will keep `details.offer_id = 0` instead of the created offer's nonzero ID from the result `OfferEntry`.

## Target Code

- `internal/transform/operation.go:701-727` — writes `details["offer_id"]` directly from the request body for manage-buy/sell offer operations
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:38742-38820` — success result exposes `OfferEntry` for created and updated offers
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:5427-5435` — `OfferEntry` carries the authoritative `OfferId`

## Evidence

The transform code never reads the manage-offer success payload before exporting `offer_id`. Checked-in operation goldens already show successful `manage_sell_offer` rows with `offer_id: 0` (`testdata/operations/large_range_ops.golden:4-5`), proving the request sentinel is currently reaching exported output.

## Anti-Evidence

The current fixtures and upstream Horizon-style processors appear to mirror this request-side behavior, so final review may decide the row is intentionally API-compatible rather than locally corrupted. But the XDR success payload already contains a more specific identifier, and exporting the create sentinel on success makes it impossible to join the operation row directly to the created offer.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the `offer_id` assignment in `extractOperationDetails()` at `internal/transform/operation.go:707,726` which reads directly from `op.OfferId` for both `ManageBuyOffer` and `ManageSellOffer`. Then compared against Horizon's canonical operations processor at `services/horizon/internal/ingest/processors/operations_processor.go:407-435`, which uses the identical pattern: `details["offer_id"] = op.OfferId` from the request body, never consulting the result. Both systems export the request-side sentinel `0` for create-new-offer operations.

### Code Paths Examined

- `internal/transform/operation.go:701-727` — stellar-etl sets `details["offer_id"] = int64(op.OfferId)` from the request body for ManageBuyOffer (line 707) and ManageSellOffer (line 726), without consulting the operation result
- `stellar/go/services/horizon/internal/ingest/processors/operations_processor.go:407-424` — Horizon's canonical processor uses the identical pattern: `details["offer_id"] = op.OfferId` from the request body for both operation types
- `testdata/operations/large_range_ops.golden:4-5` — golden fixtures confirm successful `manage_sell_offer` rows with `offer_id: 0`, consistent with the upstream behavior

### Why It Failed

This is **working-as-designed behavior**, not a bug. Horizon — the canonical Stellar API server — deliberately exports `offer_id` from the request body, meaning `0` for create-new-offer operations. stellar-etl correctly mirrors this established API contract. The `offer_id` in operation details documents **what the submitter requested**, not the outcome. The operation details field `offer_id` has always represented the request parameter across the entire Stellar ecosystem. Changing this would break API compatibility with Horizon and downstream consumers that depend on this semantic.

### Lesson Learned

When stellar-etl behavior exactly mirrors Horizon's canonical operations processor, the behavior is intentional API compatibility, not a local data corruption bug. Always compare against the upstream Horizon processor before concluding a field mapping is wrong.
