# H003: Native SAC instance rows export unsupported lumens asset metadata

**Date**: 2026-04-11
**Subsystem**: data-integrity (transform)
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-contract-data` encounters the native-asset contract's persistent instance row, it should leave `contract_data_asset_type`, `contract_data_asset_code`, and `contract_data_asset_issuer` empty. The reference SAC parser explicitly treats lumens asset stats as unsupported and does not synthesize a native asset descriptor from contract storage.

## Mechanism

`AssetFromContractData()` special-cases `sym == "Native"` and returns `xdr.MustNewNativeAsset()` whenever the contract ID matches the native asset contract. `TransformContractData()` then extracts and exports `contract_data_asset_type = "native"` from that synthetic asset, even though the upstream helper rejects native-asset contract rows entirely and the rest of the local contract-data balance path does not provide a consistent lumens asset-stat model.

## Trigger

Export contract-data changes across a ledger containing the native asset contract's persistent contract-instance row with `AssetInfo = ["Native", ...]`.

## Target Code

- `internal/transform/contract_data.go:84-90` — `TransformContractData()` copies any parsed asset into the `contract_data_asset_*` columns.
- `internal/transform/contract_data.go:191-247` — `AssetFromContractData()` recognizes `"Native"` and returns a native asset for the native contract ID.

## Evidence

The local helper explicitly branches on `case "Native"` at `contract_data.go:243-247` and returns a native asset object. The upstream implementation at `go-stellar-sdk/ingest/sac/contract_data.go:90-94` rejects the same contract ID with the comment `we don't support asset stats for lumens`.

## Anti-Evidence

This may be an intentional local extension if stellar-etl wants a partial native-asset view that upstream SAC code omits. But there is no evidence of a fully supported lumens stats contract in the surrounding transform code, which makes the one-off native asset export path look more like an accidental fork drift than a documented schema choice.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

The hypothesis claims the `case "Native"` branch at line 243 of `AssetFromContractData()` would cause native asset metadata to be exported. However, this branch is dead code that can never be reached. The Soroban encoding for `AssetInfo::Native` is a **1-element** vector `[Sym("Native")]`, but the guard at line 231 requires exactly 2 elements (`len(*vecPtr) != 2`), causing an early `nil` return before the switch statement is reached.

### Code Paths Examined

- `internal/transform/contract_data.go:230-233` — Vector length guard requires exactly 2 elements; native AssetInfo has 1 element, so returns nil before reaching the switch
- `internal/transform/contract_data.go:240-250` — The `case "Native"` branch is dead code; unreachable for actual on-chain data
- `go-stellar-sdk/ingest/sac/contract_data.go:312-318` — Upstream `metadataObjFromAsset` confirms native AssetInfo is encoded as a 1-element vector `[Sym("Native")]`
- `go-stellar-sdk/ingest/sac/contract_data.go:117-132` — Upstream switch only handles `AlphaNum4`/`AlphaNum12` with no `Native` case, consistent with the 2-element assumption

### Why It Failed

The `case "Native"` branch at line 243 is **unreachable dead code**. The Soroban `AssetInfo` enum encodes `Native` as a unit variant — a 1-element ScVec `[Sym("Native")]`. The length check at line 231 (`len(*vecPtr) != 2`) rejects this before the switch is reached. Therefore, `AssetFromContractData` always returns `nil` for native asset contract instance rows, and `TransformContractData` correctly leaves the `contract_data_asset_*` columns empty. The claimed data corruption cannot occur.

### Lesson Learned

When analyzing switch/case branches for reachability, always check upstream guard conditions first. A branch may exist in the code but be gated by an earlier validation that makes it unreachable for the data formats it's designed to handle. The Soroban contracttype enum encoding distinguishes unit variants (1-element vectors) from data-carrying variants (2-element vectors), and length checks implicitly filter between them.
