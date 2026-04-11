# H009: Asset export should include every asset-bearing operation type

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Low
**Impact**: Suspected omission of asset rows from unsupported operation types
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_assets` were intended to enumerate every asset referenced anywhere on-chain, it should recognize all asset-bearing operation types such as path payments, manage-buy offers, and trustline-related operations instead of stopping at a narrower subset.

## Mechanism

At first glance, `GetPaymentOperations()` only collecting `Payment` and `ManageSellOffer` operations looks like a data-loss bug because other valid Stellar operations can also carry non-native assets. If the command's contract were "all assets seen in the range," those omitted operation families would silently suppress valid asset rows.

## Trigger

Run `stellar-etl export_assets` on a range where the only non-native asset reference appears in an operation type other than `Payment` or `ManageSellOffer`, such as a path payment or manage-buy offer.

## Target Code

- `internal/input/assets.go:40-50` — selects only `Payment` and `ManageSellOffer`
- `internal/transform/asset.go:17-20` — rejects every other operation type
- `internal/transform/asset_test.go:30-64` — tests only the narrow accepted contract
- `README.md:236-245` — documents `export_assets` as exporting assets created from payment operations

## Evidence

The reader and transformer both hard-code a narrow operation-type filter, so a broader "all asset-bearing operations" interpretation would indeed miss rows. This was a plausible angle because the exported table is named `assets`, not `payment_assets`.

## Anti-Evidence

The repository's own contract is narrower than the hypothesis assumes: the README describes `export_assets` in payment-operation terms, and `TransformAsset()` intentionally errors on any operation outside the accepted subset. The in-tree tests also only assert payment/native cases rather than comprehensive asset discovery.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is a scope theory, not a concrete correctness bug. The codebase intentionally defines `export_assets` as a narrow export over payment/manage-sell-derived asset appearances rather than as an exhaustive inventory of every asset-bearing operation type.

### Lesson Learned

Before treating a filter as data loss, confirm the command's documented output contract and the transformer's accepted input domain. A narrow reader is only a bug when it contradicts the export's stated scope, not when it matches an intentionally limited table definition.
