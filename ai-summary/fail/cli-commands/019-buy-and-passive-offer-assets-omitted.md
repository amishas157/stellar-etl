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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of 017-path-payment-assets-omitted and 018-manage-sell-offer-buying-asset-omitted
**Failed At**: reviewer

### Trace Summary

Traced the `export_assets` pipeline through both input readers (`internal/input/assets.go:42`, `internal/input/assets_history_archive.go:32`) and the transformer (`internal/transform/asset.go:18`). All three layers consistently filter for exactly `OperationTypePayment` and `OperationTypeManageSellOffer`. This is the same intentional limited-scope design already established by reviews 017 and 018.

### Code Paths Examined

- `internal/input/assets.go:42` — Filter: `op.Body.Type == xdr.OperationTypePayment || op.Body.Type == xdr.OperationTypeManageSellOffer`
- `internal/input/assets_history_archive.go:32` — Identical filter in archive-backed reader
- `internal/transform/asset.go:18` — Guard rejects all types except `Payment` and `ManageSellOffer`

### Why It Failed

This is a **duplicate** of two previously investigated hypotheses that already addressed the exact operation types named here:

1. **017-path-payment-assets-omitted** explicitly noted: "The filter also excludes `ManageBuyOffer`, `CreatePassiveSellOffer`, `ChangeTrust`, and many other operation types that reference assets. If the exclusion of path payments were a bug, these other exclusions would be bugs too — but the command was never designed for comprehensive asset discovery across all operation types."

2. **018-manage-sell-offer-buying-asset-omitted** explicitly noted: "If extracting only Selling from ManageSellOffer is a bug, then by the same logic: not extracting from `ManageBuyOffer`, `CreatePassiveSellOffer`, `ChangeTrust`, and all path-payment types would also be bugs. The 017 review already rejected this reasoning."

Both prior reviews established that `export_assets` has an intentionally limited scope — consistently implemented across three code layers and documented in the README. The argument that "ManageSellOffer inclusion implies ManageBuyOffer/CreatePassiveSellOffer should also be included" was already considered and rejected as describing a feature limitation, not a data correctness bug.

### Lesson Learned

Before hypothesizing about missing operation types in `export_assets`, check whether prior reviews have already established the intentional-scope precedent. The 017 and 018 reviews definitively settled that the two-type filter (`Payment` + `ManageSellOffer`) is a deliberate design choice, not an omission.
