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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the complete path from `export_assets` command through `GetPaymentOperations()` → `TransformAsset()` → `transformSingleAsset()`. The code unconditionally processes all assets from payment and manage_sell_offer operations, including native XLM. The unit test at `asset_test.go:74-122` explicitly constructs a native payment input and asserts that a native asset row (`AssetType: "native"`, empty code/issuer) appears in the output. The `seenIDs` dedup map in `export_assets.go:40-58` treats native the same as any other asset. No guard against native exists anywhere in the pipeline because none was intended.

### Code Paths Examined

- `internal/transform/asset.go:14-53` — `TransformAsset()` accepts payment ops and extracts `PaymentOp.Asset` with no native filter
- `internal/transform/asset.go:55-70` — `transformSingleAsset()` calls `asset.Extract()` which handles native as `asset_type="native"` with empty code/issuer
- `internal/transform/asset_test.go:74-97` — Test input explicitly includes a native XLM payment operation
- `internal/transform/asset_test.go:104-122` — Expected output explicitly asserts native asset row is emitted
- `cmd/export_assets.go:40-58` — `seenIDs` dedup treats native identically to issued assets; no special-casing
- `internal/input/assets.go:20-61` — Reader collects all payment/manage_sell_offer ops without asset-type filtering

### Why It Failed

**Working-as-designed behavior.** The unit test was deliberately written to verify that native XLM payments produce native asset rows in the output. The `transformSingleAsset` helper intentionally handles native assets without any guard or warning. Previous fail investigations in this subsystem (009, 017) and meta-pattern 6 from the fail summary all confirm that `export_assets` is an intentional asset-reference export over payment operations — not a "newly created asset" registry. The "created" language in the README and comments is imprecise documentation, but the actual implementation is tested and intentional. A documentation wording issue is not a data correctness bug.

### Lesson Learned

When a unit test explicitly constructs a specific input scenario and asserts the corresponding output, that behavior is intentional by definition. Documentation using imprecise language ("created") does not override explicit test assertions. Documentation wording mismatches are style/clarity issues, not data correctness bugs within the scope of this investigation.
