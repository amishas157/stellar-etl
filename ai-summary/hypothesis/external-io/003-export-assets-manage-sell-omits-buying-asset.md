# H003: `export_assets` only discovers the selling side of `manage_sell_offer`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Structural data corruption: offer-based asset discovery can miss legitimate asset rows
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Once `manage_sell_offer` is included as an asset-discovery source, a successful offer should be able to contribute both assets named by that operation. A range where asset `Y` only appears as the buying side of an in-scope `manage_sell_offer` should still export an asset row for `Y`, because the command has already decided that this offer type participates in `history_assets` discovery.

## Mechanism

The input readers admit the entire `manage_sell_offer` operation, but `TransformAsset()` narrows that operation to `opSellOf.Selling` and never emits `opSellOf.Buying`. As a result, an asset that enters the selected range only as the quote/buying side of included sell-offer activity is silently omitted from `history_assets`, even though the same operation would export the base/selling asset from the identical ledger event.

## Trigger

1. Find or construct a ledger range where a successful `manage_sell_offer` references assets `X` and `Y`, and `Y` does not appear anywhere else in-range as a payment asset or as the selling side of another included offer.
2. Run `stellar-etl export_assets --start-ledger <S> --end-ledger <E> -o /tmp/assets.json`.
3. Inspect the exported asset rows.
4. The command will export `X` from the offer but omit `Y`, even though both assets were referenced by the included operation.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:40-49` — treats the full `manage_sell_offer` operation as an in-scope asset-discovery input.
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:30-39` — repeats the same offer admission on the history-archive path.
- `internal/transform/asset.go:17-38` — `ManageSellOffer` branch extracts only `opSellOf.Selling` and never visits `opSellOf.Buying`.
- `cmd/export_assets.go:44-69` — writes exactly one transformed asset row per admitted operation before deduplicating by `asset_id`.

## Evidence

The transform layer is one-sided by construction: the `ManageSellOffer` arm binds one local `asset` variable and assigns only the selling asset into it. There is no sibling append or second transform call for the buying asset anywhere in `export_assets`, so the omission persists all the way to output.

## Anti-Evidence

The overall `export_assets` command is intentionally narrow and not an exhaustive asset inventory, so a reviewer may argue that the exact offer-side semantics are part of that undocumented scope. But the code has already crossed the boundary from pure payment-based discovery into offer-based discovery by admitting `manage_sell_offer`; once that choice is made, exporting only one leg of the included offer is an internal inconsistency that can change which asset rows appear for legitimate market activity.
