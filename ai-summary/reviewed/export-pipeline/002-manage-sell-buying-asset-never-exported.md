# H002: `export_assets` drops the buy-side asset from `ManageSellOffer` discovery

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Once the asset export pipeline chooses to treat `ManageSellOffer` as an
asset-bearing operation, the exported row should surface the asset that the
operation newly introduces to the range. A sell offer that sells native XLM in
order to buy a new issued asset should not emit only the already-known native
asset and silently miss the novel buy-side asset.

## Mechanism

Both asset readers explicitly admit `ManageSellOffer` operations, but
`TransformAsset()` always selects `opSellOf.Selling` and never inspects
`opSellOf.Buying`. That means a valid range where the only first-seen reference
to asset B appears on the buy side of a sell offer will export either asset A
(the selling side) or nothing at all after `seenIDs` deduplication, even though
asset B is the new asset the offer introduces.

## Trigger

1. Find or construct a `ManageSellOffer` where:
   1. `Selling = native` (or some already-seen asset A)
   2. `Buying =` a distinct issued asset B not otherwise referenced earlier in the range
2. Run `stellar-etl export_assets` over a range that contains this offer but no
   earlier payment/offer referencing asset B.
3. The export emits native/A or skips the row after dedupe, while asset B never
   appears in `history_assets`.

## Target Code

- `internal/input/assets.go:40-49` — datastore reader intentionally includes `ManageSellOffer` in asset discovery
- `internal/input/assets_history_archive.go:30-39` — history-archive reader includes the same operation type
- `internal/transform/asset.go:17-36` — `ManageSellOffer` branch hard-codes `asset = opSellOf.Selling`
- `cmd/export_assets.go:53-58` — post-transform dedupe can suppress the already-known selling asset and leave no row for the new buy-side asset

## Evidence

The current pipeline clearly treats `ManageSellOffer` as in-scope input: both
readers append those operations, and `TransformAsset()` has an explicit
`ManageSellOffer` branch. Within that active path, however, only one of the two
referenced assets is exported. For offers that sell a common asset in order to
acquire a new one, the only asset written is the already-known selling side.

## Anti-Evidence

The README still describes `export_assets` in payment-operation terms, so a
reviewer may argue that the entire `ManageSellOffer` branch is legacy or broader
than the intended contract. Even under that interpretation, the current code is
internally inconsistent: if `ManageSellOffer` stays enabled, the one-sided
`Selling` extraction is an observable data-loss choice.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Both asset readers (`GetPaymentOperations` in `assets.go:42` and `GetPaymentOperationsHistoryArchive` in `assets_history_archive.go:32`) filter for `OperationTypeManageSellOffer`, feeding each matching operation to `TransformAsset()`. The transform function's `ManageSellOffer` branch (asset.go:24-29) unconditionally assigns `asset = opSellOf.Selling` and returns a single `AssetOutput`. The `Buying` field of the operation is never accessed. The export command's `seenIDs` dedupe map (export_assets.go:40-58) then suppresses already-seen selling-side assets, but cannot compensate for a buying-side asset that was never extracted. The function signature (`TransformAsset → AssetOutput`) structurally returns one asset per call, so even the caller loop cannot produce two rows from one operation.

### Code Paths Examined

- `internal/input/assets.go:42` — datastore reader includes `OperationTypeManageSellOffer` in the filter predicate
- `internal/input/assets_history_archive.go:32` — history-archive reader includes the same filter predicate
- `internal/transform/asset.go:24-29` — `ManageSellOffer` case extracts only `opSellOf.Selling`; `opSellOf.Buying` is never referenced
- `internal/transform/asset.go:14` — function returns a single `AssetOutput`, not a slice
- `cmd/export_assets.go:44-56` — caller iterates one transform result per operation; dedupe map can only suppress, not add, rows

### Findings

The bug is confirmed through direct code reading:

1. **Both readers admit ManageSellOffer** — `assets.go:42` and `assets_history_archive.go:32` both check `op.Body.Type == xdr.OperationTypeManageSellOffer`.

2. **Transform extracts only the selling side** — `asset.go:29` assigns `asset = opSellOf.Selling`. The `Buying` field from `ManageSellOfferOp` is never accessed anywhere in the function.

3. **Single-return architecture prevents implicit fix** — `TransformAsset` returns `(AssetOutput, error)`, not a slice. Even if a caller wanted both assets, the current signature cannot deliver them.

4. **Dedupe amplifies the loss** — When the selling asset is already known (common for native XLM), the `seenIDs` check at `export_assets.go:54` skips the row entirely. The buying-side asset was never extracted, so it produces zero rows total for that operation.

5. **Neither `ManageBuyOffer` nor `PathPayment*` are in scope** — The filter at `asset.go:18` rejects everything except `Payment` and `ManageSellOffer`. This is not a broader asset-discovery tool; the partial ManageSellOffer support is specifically what creates the inconsistency.

**Impact**: Any asset whose first (or only) appearance in an exported ledger range is on the buying side of a `ManageSellOffer` will be silently absent from the `history_assets` export. Downstream BigQuery tables that join on `asset_id` will have no matching row for that asset.

### PoC Guidance

- **Test file**: `internal/transform/asset_test.go` (or create a new test in `cmd/` if integration-level)
- **Setup**: Construct a `ManageSellOfferOp` with `Selling = native` and `Buying = CreditAlphanum4{AssetCode: "TEST", Issuer: <some account>}`. Wrap it in an `xdr.Operation` with `Body.Type = OperationTypeManageSellOffer`.
- **Steps**: Call `TransformAsset(op, 0, 0, 100, lcm)` and inspect the returned `AssetOutput`.
- **Assertion**: Assert that the returned `AssetOutput.AssetCode` equals `"TEST"` (the buying asset). The current code will return the native asset instead, confirming the bug. Alternatively, demonstrate that calling `TransformAsset` once per operation can never yield both assets — showing the structural gap.
