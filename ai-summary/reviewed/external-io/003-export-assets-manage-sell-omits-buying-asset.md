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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full export_assets pipeline from input filtering through transform to output. Both `GetPaymentOperations` (assets.go:42) and `GetPaymentOperationsHistoryArchive` (assets_history_archive.go:32) admit `OperationTypeManageSellOffer` into the processing scope alongside `OperationTypePayment`. However, `TransformAsset` (asset.go:24-29) extracts only `opSellOf.Selling` from the ManageSellOffer body — there is no code path for `opSellOf.Buying`. The export command (export_assets.go:44-69) calls `TransformAsset` once per operation and writes one row, with no mechanism to emit multiple assets per operation.

### Code Paths Examined

- `internal/input/assets.go:42` — `op.Body.Type == xdr.OperationTypeManageSellOffer` admits the full operation
- `internal/input/assets_history_archive.go:32` — identical filter on the history-archive path
- `internal/transform/asset.go:24-29` — `case xdr.OperationTypeManageSellOffer:` extracts only `opSellOf.Selling`, `opSellOf.Buying` is never read
- `internal/transform/asset.go:40-52` — `transformSingleAsset(asset)` processes only the single selling asset
- `cmd/export_assets.go:44-69` — loop calls `TransformAsset` once per admitted operation, producing one row; `seenIDs` dedup is correct but irrelevant to the omission
- `internal/transform/asset_test.go:74-97` — tests only cover Payment operations, no ManageSellOffer test coverage

### Findings

The asymmetry is confirmed by direct code reading. The ManageSellOffer operation carries two assets (`Selling` and `Buying`), but only `Selling` is extracted. The buying asset is silently dropped. This means an asset appearing exclusively as the buying/quote side of ManageSellOffer operations within a given ledger range will produce no row in the export.

Severity is downgraded from High to Medium because:
1. The `export_assets` command is documented as intentionally narrow ("Exports the assets that are created from payment operations" — README:244, cmd/export_assets.go:17)
2. The function is named `GetPaymentOperations`, framing the scope around "payment-like" asset extraction
3. The selling asset is the "outgoing/payment" analog for offers, so there is a plausible (though undocumented) design rationale
4. The practical impact requires a specific scenario: an asset appearing ONLY as the buying side of ManageSellOffer in the entire exported range

Despite these mitigating factors, the inconsistency is real: the code admits the operation but only processes half its data. This is a conditional row omission, not a design choice that's been explicitly justified anywhere.

### PoC Guidance

- **Test file**: `internal/transform/asset_test.go`
- **Setup**: Construct a `ManageSellOfferOp` with a known `Selling` asset (e.g., USDT) and a known `Buying` asset (e.g., EUR credit). Wrap it in an `xdr.Operation` with `OperationTypeManageSellOffer`.
- **Steps**: Call `TransformAsset()` with the constructed operation. Verify it returns only the selling asset. Then demonstrate that no call path in the export pipeline ever processes the buying asset — there is no second call to `TransformAsset` or `transformSingleAsset` for the buying side.
- **Assertion**: Assert that `TransformAsset` returns only USDT (the selling asset) and that EUR (the buying asset) is not exported anywhere. This demonstrates the asymmetric extraction.
