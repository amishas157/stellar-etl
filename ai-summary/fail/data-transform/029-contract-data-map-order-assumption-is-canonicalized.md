# H029: Contract-data balance parsing depends on fixed `ScMap` entry order

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`AssetFromContractData()` and `ContractBalanceFromContractData()` should recognize valid SAC asset-info and balance rows regardless of how the underlying `ScMap` entries were inserted by contract code. If the same key/value pairs are present, map-entry ordering should not determine whether `asset_*`, `balance_holder`, or `balance` fields are exported.

## Mechanism

The transform code indexes directly into `assetMap[0]`, `assetMap[1]`, and `balanceMap[0..2]` instead of searching by key, which initially looks like an order-dependent parser that could silently reject valid rows. That would matter because rejected SAC balance rows leave the exported contract-data asset and balance columns empty.

## Trigger

Construct a valid SAC `ContractData` row whose `ScMap` entries contain the same keys as today but appear in a different order.

## Target Code

- `internal/transform/contract_data.go:253-281` — asset-info parser assumes `asset_code` then `issuer`
- `internal/transform/contract_data.go:345-366` — balance parser assumes `amount`, `authorized`, `clawback`

## Evidence

Both parsers use positional indexing into `ScMap` slices rather than iterating and matching keys, so the code would reject differently ordered maps. At first glance, that looks like a brittle transform boundary for contract storage.

## Anti-Evidence

Soroban `SCMap` values are canonically ordered by key before serialization, so legitimate on-chain rows do not arrive in arbitrary insertion order. The specific key names used here also sort into the exact order the parser expects (`amount` < `authorized` < `clawback`, `asset_code` < `issuer`).

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The transform's positional map indexing is brittle in theory, but Soroban's canonical key ordering makes the hypothesized alternate ordering unreachable for legitimate ledger data. Because valid `SCMap` serialization already arrives in the same key order this parser assumes, the transform does not currently lose rows on this path.

### Lesson Learned

Order-sensitive `ScMap` parsing is only a live bug if the protocol allows non-canonical entry ordering. For Soroban contract-data hypotheses, verify host/protocol ordering guarantees before treating positional map access as an actual data-loss path.
