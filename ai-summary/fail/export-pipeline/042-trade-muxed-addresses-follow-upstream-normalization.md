# H042: Trade addresses flatten muxed subaccounts into base `G...` accounts

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: counterparty identity mismatch
**Hypothesis by**: GPT-5.4, high

## Expected Behavior

If `history_trades` is supposed to preserve the exact buyer and seller identities for muxed-account trades, then a trade whose buyer or seller is a muxed `M...` account should export data that keeps that subaccount distinct from the underlying base `G...` account.

## Mechanism

`TransformTrade()` looked suspicious because it resolves the seller with `claimOffer.SellerId().Address()` and resolves the buyer through `.ToAccountId().Address()`, which appears to flatten muxed identities to their base account form. That would silently merge distinct muxed subaccounts into the same exported trade counterparty.

## Trigger

Export trades for a ledger containing a manage-offer or path-payment trade where either the offer seller or the operation source account is muxed.

## Target Code

- `internal/transform/trade.go:111-128` — trade export derives seller/buyer addresses with base-account helpers
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/trade/trade.go:133-152` — upstream Horizon-style trade processor uses the same normalization

## Evidence

The local trade transformer has no muxed-specific sibling fields and explicitly converts the buyer through `ToAccountId().Address()`. On first read, that looks like silent loss of muxed subaccount identity.

## Anti-Evidence

The upstream `stellar/go` trade processor uses the exact same address derivation, and the shared trade schema only exposes `selling_account_address` / `buying_account_address` base-address columns. I found no local contract, schema field, or existing test that promises muxed-preserving trade output.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is upstream-mirroring schema behavior, not a local ETL divergence. `history_trades` only defines base-address columns, and the local implementation matches the upstream `stellar/go` trade processor exactly, so there is no codebase-specific bug to fix in `stellar-etl`.

### Lesson Learned

For identity-loss candidates, first check whether the table schema even has a place to preserve the richer representation. If the local code matches the upstream Horizon-style processor and the schema only exposes base-address columns, muxed flattening is a product-contract question, not a novel local integrity bug.
