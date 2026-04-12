# H002: Asset discovery omits `create_passive_sell_offer` even though it already includes the same sell-offer family via `manage_sell_offer`

**Date**: 2026-04-12
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `history_assets` is populated from offer-derived asset discovery at all, a `create_passive_sell_offer` should contribute its selling asset exactly like `manage_sell_offer` does. A ledger range where an issued asset only first appears through a passive sell offer should still export that asset row.

## Mechanism

The asset readers and `TransformAsset` hardcode `ManageSellOffer` as the only offer operation that can "issue" an asset, even though `CreatePassiveSellOffer` is the legacy sell-offer variant over the same selling asset concept. As a result, an asset introduced solely through passive-sell-offer activity is silently omitted from `export_assets`, while the equivalent modern sell-offer path is included.

## Trigger

1. Find a ledger range containing a `create_passive_sell_offer` for an issued asset that does not also appear in a `payment` or `manage_sell_offer` within the same export range.
2. Run `stellar-etl export_assets --start-ledger <S> --end-ledger <E>`.
3. Inspect `history_assets`.
4. The asset is omitted even though the same selling asset would be exported if the offer were encoded as `manage_sell_offer`.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:41-49` — filters only `payment` and `manage_sell_offer`
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:31-39` — repeats the same enum filter for history-archive exports
- `internal/transform/asset.go:17-38` — `TransformAsset` rejects every operation type except `payment` and `manage_sell_offer`
- `internal/input/trades.go:58-60` — sibling reader comment treats `manage sell offer` and `create passive sell offer` as the same trade-producing offer family

## Evidence

The omission is a strict whitelist bug: neither reader admits `CreatePassiveSellOffer`, and the transformer would reject it even if the input layer passed it through. Elsewhere in the codebase, passive sell offers are already treated as a first-class sell-offer operation rather than an unrelated type, which makes this asset-reader gap look like an inconsistent partial implementation inside the same offer family.

## Anti-Evidence

Prior investigations have established that `export_assets` is intentionally narrow and not an exhaustive asset inventory. A reviewer may therefore conclude that the command only wants `payment` plus the one specific offer opcode already wired today. The counterpoint is that once `manage_sell_offer` is already in scope, omitting its legacy passive-sell sibling is no longer a clean scope boundary; it is a family-level inconsistency that changes exported rows for legitimate classic-offer ledgers.
