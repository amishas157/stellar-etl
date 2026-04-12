# H003: Invoke-contract balance-change details mix raw muxed and canonical account identities

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: operation-detail identity corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `export_operations` rows with `details.type="invoke_contract"`, the nested `details.asset_balance_changes[*].from` / `to` account fields should use the same identity convention as the rest of the ETL's account surfaces: canonical `G...` account IDs for the base field, with muxed identity preserved separately if needed. Downstream consumers should not see the same Stellar account represented as `G...` in one export surface and `M...` in another column that semantically means "account".

## Mechanism

`extractOperationDetails()` populates `asset_balance_changes` via `parseAssetBalanceChangesFromContractEvents()`, which passes the raw strings from `contractevents.NewStellarAssetContractEvent()` straight into `createSACBalanceChangeEntry()`. The SDK obtains those strings from `ScAddress.String()`, which returns `M...` for muxed accounts, and this helper never canonicalizes or splits that identity. As a result, invoke-contract balance-change details can export raw muxed addresses in `from` / `to` while sibling operation/effect helpers normalize account fields to canonical `G...` plus muxed companion metadata.

## Trigger

Run `export_operations` on a ledger containing an `invoke_contract` operation whose SAC transfer/mint/burn/clawback event references a muxed account. Inspect `details.asset_balance_changes`: the affected entry will carry `from` or `to` as an `M...` address instead of the canonical account form used elsewhere in operation exports.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:1063-1097` — wires `invoke_contract` details to `parseAssetBalanceChangesFromContractEvents()`
- `internal/transform/operation.go:parseAssetBalanceChangesFromContractEvents:1942-1975` — feeds raw SAC endpoint strings into the exported detail list
- `internal/transform/operation.go:createSACBalanceChangeEntry:1984-1998` — copies `from` / `to` directly without canonicalization or muxed metadata
- `github.com/stellar/go-stellar-sdk@v0.1.0/support/contractevents/utils.go:15-51` — source of the raw `from` / `to` strings
- `github.com/stellar/go-stellar-sdk@v0.1.0/xdr/scval.go:13-34` — muxed `ScAddress` stringification behavior

## Evidence

The local comment in `parseAssetBalanceChangesFromContractEvents()` says the parser expresses endpoints as account `G...` or contract `C...` strings, but the SDK now has an explicit muxed-account `ScAddress` arm and returns `M...` for it. Elsewhere in the transform layer, muxed-aware helpers like `addAccountAndMuxedAccountDetails()` and `addMuxed()` deliberately separate canonical and muxed identity, making this raw-string pass-through an outlier.

## Anti-Evidence

`asset_balance_changes` is a free-form JSON detail blob, not a strongly typed top-level schema, so one could argue raw `M...` values are still "exact" rather than strictly wrong. The hypothesis depends on the export contract favoring canonical account columns across surfaces, which is strongly suggested by sibling helpers but not spelled out for this nested field.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the complete code path from `ScAddress.String()` through `parseBalanceChangeEvent` to the ETL's `createSACBalanceChangeEntry`. Confirmed that `ScAddressTypeScAddressTypeMuxedAccount` (enum value 2) exists in the SDK's XDR and that `ScAddress.String()` returns `M...` for muxed accounts via `MuxedAccount.GetAddress()`. However, the upstream Horizon processor (`stellar/go@v0.0.0-20250729094549/services/horizon/internal/ingest/processors/operations_processor.go:803-843`) uses the identical pattern: its `createSACBalanceChangeEntry` copies `sacEvent.From`/`sacEvent.To` directly from `ScAddress.String()` output without canonicalization, producing the same `M...` strings for muxed accounts.

### Code Paths Examined

- `internal/transform/operation.go:1907-1940` — ETL's `parseAssetBalanceChangesFromContractEvents` calls `contractevents.NewStellarAssetContractEvent` then passes `transferEvt.From`/`.To` etc. to `createSACBalanceChangeEntry`
- `internal/transform/operation.go:1984-1998` — ETL's `createSACBalanceChangeEntry` copies `fromAccount`/`toAccount` strings directly into the map
- `go-stellar-sdk@v0.0.0-20251211085638/support/contractevents/utils.go:20-52` — `parseBalanceChangeEvent` calls `ScAddress.String()` on topics[1] and topics[2], returning `M...` for muxed accounts
- `go-stellar-sdk@v0.0.0-20251211085638/xdr/scval.go:25-34` — `ScAddress.String()` for `ScAddressTypeMuxedAccount` creates a `MuxedAccount` and calls `GetAddress()` which uses `strkey.VersionByteMuxedAccount` encoding
- `stellar/go@v0.0.0-20250729094549/services/horizon/internal/ingest/processors/operations_processor.go:803-843` — upstream Horizon's `createSACBalanceChangeEntry` copies `sacEvent.From`/`.To` directly, same pattern as ETL
- `stellar/go@v0.0.0-20250729094549/services/horizon/internal/ingest/contractevents/events.go:49-53` — upstream's `parseAddress` also calls `ScAddress.String()`, returning `M...` for muxed accounts
- `internal/transform/operation.go:423-440` — `addAccountAndMuxedAccountDetails` operates on `xdr.MuxedAccount` from tx envelopes (different type from `ScAddress` strings in contract events)

### Why It Failed

The ETL intentionally mirrors the upstream Horizon processor's behavior for `asset_balance_changes`. The upstream Horizon's `createSACBalanceChangeEntry` (operations_processor.go:803-843) copies `From`/`To` directly from `ScAddress.String()` output — the exact same pattern the ETL uses. Both produce `M...` for muxed accounts. The `addAccountAndMuxedAccountDetails` helper referenced by the hypothesis operates on `xdr.MuxedAccount` from transaction envelopes, a fundamentally different data type than the `string` outputs of `ScAddress.String()` in contract events; these are not comparable surfaces. The upstream does add a `destination_muxed_id` field derived from V4 event `to_muxed_id` data, but this is a separate feature (V4 memo extraction) not present in the old SDK the ETL uses — and it doesn't change the `from`/`to` canonicalization behavior. This is the same upstream-mirroring pattern documented in fail/cli-commands/021 for effects classification.

### Lesson Learned

The `asset_balance_changes` `from`/`to` fields follow the upstream Horizon processor's convention of passing raw `ScAddress.String()` output, not the `addAccountAndMuxedAccountDetails` canonicalization pattern used for tx-envelope accounts. These are different address-source types (`ScAddress` vs `xdr.MuxedAccount`) processed through different code paths by design. Always verify the upstream Horizon operations processor's handling of the same detail field before flagging ETL behavior as inconsistent.
