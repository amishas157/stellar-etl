# H048: Contract-data SAC parsers fail when Soroban map entries appear in a different key order

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If Soroban `ScMap` key order were non-deterministic, `AssetFromContractData()` and `ContractBalanceFromContractData()` should locate entries by key equality rather than by fixed positions so valid SAC asset-info and balance rows are exported regardless of serialization order.

## Mechanism

Both SAC helpers index directly into map slots (`assetMap[0]`, `balanceMap[0]`, etc.) before validating the expected symbol keys. If legitimate on-chain maps could serialize the same key/value pairs in a different order, the parser would silently reject real SAC rows and drop `asset_*` / `balance_*` output from `history_contract_data`.

## Trigger

1. Construct or locate a valid SAC `ContractData` row whose `ScMap` entries are the same but appear in a different physical order.
2. Run it through `TransformContractData()`.
3. Observe whether `asset_*` or `balance_*` fields disappear even though the semantic content is unchanged.

## Target Code

- `internal/transform/contract_data.go:253-279` — asset-info parser assumes `asset_code` then `issuer`.
- `internal/transform/contract_data.go:345-366` — balance parser assumes `amount`, `authorized`, then `clawback`.

## Evidence

The local parser is visibly positional: it does not search the map for matching keys. At first glance, that looks like a structural parser bug because XDR map types are often modeled as variable-length entry arrays rather than keyed lookup tables.

## Anti-Evidence

Soroban `ScMap` serialization is canonically key-sorted, and the specific symbol sets used here sort into exactly the order the parser expects (`amount < authorized < clawback`, `asset_code < issuer`). That makes the apparent position dependence stable for legitimate on-chain SAC rows rather than a live source of export drift.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected bug depends on Soroban maps preserving arbitrary insertion order, but SAC `ScMap` values are canonically serialized in sorted key order. For the concrete symbol sets used by these parsers, the canonical order matches the hardcoded slot checks, so legitimate rows do not drift.

### Lesson Learned

Position-based parsing of XDR map entries is only actionable when the underlying map type permits multiple wire orders for the same logical content. For Soroban `ScMap`, canonical key ordering can make an apparently fragile slot-based parser stable, so map-order concerns need protocol-level confirmation before they count as data corruption.
