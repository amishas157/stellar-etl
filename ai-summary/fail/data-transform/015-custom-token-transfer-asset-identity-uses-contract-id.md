# H015: Custom token-transfer rows intentionally use `contract_id`, not classic asset fields

**Date**: 2026-04-15
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A token-transfer export row should preserve enough asset identity to distinguish native assets, classic issued assets, and custom contract tokens. For custom SEP-41 transfers, that identity should come from the contract-address path rather than be fabricated as a classic asset tuple.

## Mechanism

`getAssetFromEvent()` only handles `native` and `issued_asset`, so it initially looked like custom SEP-41 token transfers would silently lose asset identity by exporting empty `asset`, `asset_type`, `asset_code`, and `asset_issuer` fields. If the upstream event model carried a third asset arm for custom tokens, this transform would be dropping it entirely.

## Trigger

Export token-transfer rows for a custom SEP-41 token contract whose events are not backed by a classic issued asset.

## Target Code

- `internal/transform/token_transfer.go:93-150` — derives exported asset fields from the event asset proto
- `github.com/stellar/go-stellar-sdk/protos/asset/asset.proto:10-16` — asset proto only defines `native` and `issued_asset`
- `internal/transform/schema.go:665-676` — token-transfer schema also exports `contract_id`

## Evidence

The transformer leaves the classic asset fields empty unless `event.GetAsset().GetNative()` or `GetIssuedAsset()` succeeds. That means custom-token rows cannot populate `asset_code` / `asset_issuer` through this helper.

## Anti-Evidence

The upstream asset proto has no third union arm for SEP-41/custom contract tokens; the only canonical non-classic identity exposed to the ETL is `event.Meta.ContractAddress`, which the transform already exports as `contract_id`. There is no hidden asset payload that the ETL is discarding.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The empty classic-asset fields are consistent with the current upstream event model. Custom-token identity is carried by `contract_id`, not by an omitted third asset variant in the proto, so the transform is not locally dropping a reachable field.

### Lesson Learned

For token-transfer hypotheses, inspect the upstream proto definitions before assuming a missing branch exists. If the event schema exposes only classic asset variants plus a separate contract identifier, an empty classic tuple on custom-token rows is a data-model decision, not proof of a transform bug.
