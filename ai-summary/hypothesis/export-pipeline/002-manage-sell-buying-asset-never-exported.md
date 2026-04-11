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
