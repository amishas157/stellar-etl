# H003: `export_assets` ignores `ManageBuyOffer` and `CreatePassiveSellOffer` assets despite already mining offer ops

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_assets` uses offer operations as asset-discovery inputs, it should treat the sibling offer operation types consistently. Assets that appear only in `ManageBuyOffer` or `CreatePassiveSellOffer` should be exported just like assets seen in `ManageSellOffer`, because all three op types carry explicit `selling` and `buying` asset fields.

## Mechanism

Both asset readers and `TransformAsset()` are hard-coded to `Payment` plus `ManageSellOffer`. `ManageBuyOffer` and `CreatePassiveSellOffer` never reach the transformer even though their XDR bodies expose the same asset-bearing fields as `ManageSellOffer`. The result is a structurally incomplete `history_assets` export whenever an asset is only surfaced through those sibling offer types in the scanned range.

## Trigger

Run `export_assets` over a range where an asset is referenced only by `ManageBuyOffer` or `CreatePassiveSellOffer` operations and does not also appear in a plain `Payment` or `ManageSellOffer`. The correct export should include that asset, but the current command omits it because the reader filter never admits those op types.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:40-49` — qualifying op-type filter excludes `ManageBuyOffer` and `CreatePassiveSellOffer`
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:30-39` — same omission in the history-archive path
- `internal/transform/asset.go:17-19` — transformer rejects op types other than `Payment` and `ManageSellOffer`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:26287-26293` — `ManageBuyOfferOp` carries both `Selling` and `Buying`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:26391-26396` — `CreatePassiveSellOfferOp` carries both `Selling` and `Buying`

## Evidence

The current code already crossed the "payment-only" boundary by including `ManageSellOffer` in both reader paths, so the omission is not just documentation drift. The sibling offer ops with the same asset-bearing structure are arbitrarily excluded by the literal op-type checks.

## Anti-Evidence

README still describes `export_assets` in payment-centric terms, so a reviewer could decide the offer-op coverage is intentionally partial. The strongest evidence against that interpretation is the code's existing `ManageSellOffer` support, which already broadens discovery beyond plain payments.
