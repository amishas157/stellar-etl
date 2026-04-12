# H003: `export_assets` emits a `history_assets` row for native XLM even though the reader is framed as discovering newly created assets

**Date**: 2026-04-12
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_assets` should only emit rows for non-native assets newly discovered through the command's qualifying operations. A range containing only native-XLM payments should not manufacture a `history_assets` row for the pre-existing native asset, because lumens are not "created" or "newly issued" by any payment operation in that range.

## Mechanism

The payment branch of `TransformAsset` blindly forwards `PaymentOp.Asset` into `transformSingleAsset`, and that helper happily serializes the native asset as `asset_type=native` with empty `asset_code` and `asset_issuer`. The asset reader comments and README describe this path as discovering new/created assets, so any ordinary XLM payment causes the exporter to output a plausible but semantically wrong `history_assets` row for an asset that was never created by the triggering operation.

## Trigger

1. Choose a ledger range containing a native-XLM `payment` but no issued-asset `payment` or `manage_sell_offer`.
2. Run `stellar-etl export_assets --start-ledger <S> --end-ledger <E>`.
3. Inspect the output.
4. The export can contain a row with `asset_type="native"` and blank code/issuer even though no new issued asset was introduced in the range.

## Target Code

- `README.md:236-244` — documents `export_assets` as exporting assets "created from payment operations"
- `internal/input/assets.go:20-21` — reader comment says payment operations "can include new assets"
- `internal/input/assets_history_archive.go:12-13` — same "can include new assets" framing on the archive path
- `internal/transform/asset.go:31-52` — payment branch exports whatever `PaymentOp.Asset` contains and stamps it into `history_assets`
- `internal/transform/asset.go:55-69` — `transformSingleAsset` explicitly accepts native and assigns it a stable `AssetID`
- `internal/transform/asset_test.go:74-97` — test input includes a native payment
- `internal/transform/asset_test.go:104-121` — expected output asserts that a native row is emitted

## Evidence

The current behavior is not hypothetical; the unit test explicitly expects a native asset row with `asset_type="native"` and blank code/issuer. That means any range containing an XLM payment will export the network-native asset into `history_assets`, even though the surrounding comments and README consistently describe this path as discovering newly created assets rather than enumerating every referenced asset kind.

## Anti-Evidence

The codebase may ultimately treat `history_assets` as an operation-derived asset-reference table rather than authoritative asset creation history, in which case including native would be a deliberate semantic choice. But the current command and reader documentation repeatedly use "created" / "new assets" / "issue an asset" language, so emitting lumens still looks like a meaningful contract mismatch rather than a harmless formatting choice.
