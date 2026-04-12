# H002: Asset discovery omits `create_passive_sell_offer` even though it already includes the same sell-offer family via `manage_sell_offer`

**Date**: 2026-04-12
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `history_assets` is populated from offer-derived asset discovery at all, a `create_passive_sell_offer` should contribute its selling asset exactly like `manage_sell_offer` does. A ledger range where an issued asset only first appears through a passive sell offer should still export that asset row.

## Mechanism

The asset readers and `TransformAsset` hardcode `ManageSellOffer` as the only offer operation that can "issue" an asset, even though `CreatePassiveSellOffer` is the legacy sell-offer variant over the same selling asset concept. As a result, an asset introduced solely through passive-sell-offer activity is silently omitted from `export_assets`, while the equivalent modern sell-offer path is included.

## Trigger

1. Find a ledger range containing a `create_passive_sell_offer` for an issued asset that does not also appear in a `payment` or `manage_sell_offer` within the same export range.
2. Run `stellar-etl export_assets --start-ledger <S> --end-ledger <E>`.
3. Inspect `history_assets`.
4. The asset is omitted even though the same selling asset would be exported if the offer were encoded as `manage_sell_offer`.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:41-49` — filters only `payment` and `manage_sell_offer`
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:31-39` — repeats the same enum filter for history-archive exports
- `internal/transform/asset.go:17-38` — `TransformAsset` rejects every operation type except `payment` and `manage_sell_offer`
- `internal/input/trades.go:58-60` — sibling reader comment treats `manage sell offer` and `create passive sell offer` as the same trade-producing offer family

## Evidence

The omission is a strict whitelist bug: neither reader admits `CreatePassiveSellOffer`, and the transformer would reject it even if the input layer passed it through. Elsewhere in the codebase, passive sell offers are already treated as a first-class sell-offer operation rather than an unrelated type, which makes this asset-reader gap look like an inconsistent partial implementation inside the same offer family.

## Anti-Evidence

Prior investigations have established that `export_assets` is intentionally narrow and not an exhaustive asset inventory. A reviewer may therefore conclude that the command only wants `payment` plus the one specific offer opcode already wired today. The counterpoint is that once `manage_sell_offer` is already in scope, omitting its legacy passive-sell sibling is no longer a clean scope boundary; it is a family-level inconsistency that changes exported rows for legitimate classic-offer ledgers.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not a duplicate (fail 009 targeted path-payment/manage-buy/trustline, not create_passive_sell_offer), but the same by-design principle applies
**Failed At**: reviewer

### Trace Summary

I traced the complete asset export pipeline: `cmd/export_assets.go` → `input.GetPaymentOperations()` / `input.GetPaymentOperationsHistoryArchive()` → `transform.TransformAsset()`. All three layers explicitly whitelist exactly `xdr.OperationTypePayment` and `xdr.OperationTypeManageSellOffer`. The documentation in the README ("Exports the assets that are created from payment operations"), the cobra Long description ("Exports the assets that are created from payment operations over a specified ledger range"), and the transformer's explicit error on all other types confirm this is an intentionally scoped export, not an incomplete whitelist.

### Code Paths Examined

- `internal/input/assets.go:42` — `op.Body.Type == xdr.OperationTypePayment || op.Body.Type == xdr.OperationTypeManageSellOffer` — explicit two-type whitelist
- `internal/input/assets_history_archive.go:32` — identical whitelist in history-archive reader
- `internal/transform/asset.go:18` — `if opType != xdr.OperationTypePayment && opType != xdr.OperationTypeManageSellOffer` — transformer rejects all other types with a clear error message
- `cmd/export_assets.go:17` — Long description: "Exports the assets that are created from payment operations over a specified ledger range"
- `README.md:244` — "Exports the assets that are created from payment operations over a specified ledger range"
- `internal/input/trades.go:58-60` — comment acknowledges create_passive_sell_offer in the trade family, but this is a different export with different scope

### Why It Failed

The `export_assets` command has an explicitly defined scope: `Payment` + `ManageSellOffer`. This is documented in the README, the cobra command description, and enforced at both the input and transform layers. Prior investigation (fail 009) established the principle that this narrow scope is intentional ("a narrow reader is only a bug when it contradicts the export's documented scope"). The fact that `create_passive_sell_offer` is semantically close to `manage_sell_offer` does not make the omission a bug — the scope boundary is "these two specific operation types," not "the sell-offer operation family." Adding `CreatePassiveSellOffer` would be a feature enhancement, not a correctness fix. Meta-pattern #6 in the fail summary explicitly codifies this: "Intentional command scope: export_assets is intentionally a narrow payment/manage-sell-operation asset-reference export, not an exhaustive asset inventory."

### Lesson Learned

The family-consistency argument ("if manage_sell_offer is included, create_passive_sell_offer should be too") is more compelling than the broad-scope argument rejected in fail 009, but it still runs into the same fundamental design decision. The scope of `export_assets` is defined by explicit operation type checks, not by semantic operation families. When the code, documentation, and prior investigations all agree on intentional narrowness, a sibling-operation omission is a feature gap, not a data correctness bug.
