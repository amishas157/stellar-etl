# H013: Custom token transfer asset fields should be backfilled from `contract_id`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

At first glance, custom-token `TokenTransferOutput` rows seemed like they should populate `asset` and `asset_type` with some contract-derived identifier instead of leaving those fields empty. A row describing a token movement should ideally preserve a non-empty asset identity across all token classes.

## Mechanism

`getAssetFromEvent()` only understands native and issued Stellar assets, so when the upstream SDK omits `event.GetAsset()` for custom tokens the transformer emits empty `asset`, `asset_type`, `asset_code`, and `asset_issuer` values. That looked like a likely data-loss bug because the row still carries a populated `contract_id`.

## Trigger

Process a non-SAC custom token event through `export_token_transfer`. The exported row contains `contract_id`, but the classic asset fields remain empty.

## Target Code

- `internal/transform/token_transfer.go:getAssetFromEvent:132-150` — only native and issued assets populate output asset fields
- `internal/transform/token_transfer.go:transformEvents:93-125` — appends empty asset fields while still exporting `contract_id`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/processors/token_transfer/token_transfer_event.pb.go:124-129` — SDK `Transfer.Asset` comment says custom-token events intentionally omit `Asset`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/processors/token_transfer/token_transfer_event.pb.go:38-45` — event metadata separately carries `ContractAddress`

## Evidence

The upstream SDK contract explicitly documents that `Transfer.Asset` / `Mint.Asset` / related event assets are absent for custom tokens, while `EventMeta.ContractAddress` carries the contract identity. The local transformer mirrors that split by leaving classic asset fields empty and exporting `ContractID: eventMeta.ContractAddress`.

## Anti-Evidence

The exported schema already has a dedicated `contract_id` field, and the upstream token-transfer processor deliberately uses that field as the identity bridge for non-classic tokens. That makes the empty classic-asset columns look like intentional schema partitioning rather than silent corruption.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The upstream token-transfer model intentionally omits `Asset` for custom tokens and provides `contract_address` instead. Stellar-etl already preserves that contract identifier as `contract_id`, so leaving classic asset columns empty for non-native/non-issued tokens is consistent with the source model rather than a transform bug.

### Lesson Learned

For token-transfer rows, `asset` / `asset_type` are classic-asset descriptors, while `contract_id` is the authoritative identity for custom tokens. Empty classic-asset fields are not automatically corruption when the SDK explicitly documents that custom-token events omit `Asset`.
