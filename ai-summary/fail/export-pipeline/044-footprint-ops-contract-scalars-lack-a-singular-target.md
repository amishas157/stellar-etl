# H044: Footprint TTL operations misidentify a single target contract from multi-key footprints

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: misleading operation details
**Hypothesis by**: GPT-5.4, high

## Expected Behavior

If `extend_footprint_ttl` or `restore_footprint` export scalar `contract_id` / `contract_code_hash` detail fields, those scalars should identify the actual operation target rather than whichever contract or code hash happens to appear first in the transaction footprint.

## Mechanism

The transform builds these details by scanning the whole Soroban footprint and returning the first matching contract-data or contract-code key. That looks order-dependent and potentially wrong when a single footprint operation covers multiple ledger keys from different contracts.

## Trigger

Export operations for a footprint-extension or restore transaction whose Soroban footprint contains multiple contract-data and/or contract-code keys from different contracts.

## Target Code

- `internal/transform/operation.go:1144-1159` — `extend_footprint_ttl` / `restore_footprint` populate scalar contract fields from footprint helpers
- `internal/transform/operation.go:1808-1857` — footprint helpers walk the full transaction footprint and return the first matching contract ID / code hash
- `internal/transform/operation_test.go:2075-2119` — existing tests treat these scalar fields as optional summaries and accept empty values

## Evidence

The helper functions are explicitly first-match scans over `ReadWrite` and `ReadOnly`, so their outputs are order-sensitive by construction. For a heterogeneous footprint, that makes the scalar contract fields look like plausible but arbitrary summaries.

## Anti-Evidence

These operations fundamentally target sets of ledger keys, and the same detail payload already exports `ledger_key_hash` as the authoritative multi-key list. I found no schema contract, upstream invariant, or repository test that defines which single contract or code hash must represent a heterogeneous footprint.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Without a documented singular-target rule, I cannot prove that the first matching contract/hash is incorrect rather than an intentionally lossy summary for a multi-key operation. The operation already exports the full `ledger_key_hash` list, so the scalar fields are not authoritative enough to support a concrete corruption claim.

### Lesson Learned

For multi-key Soroban operations, a suspicious scalar summary is only a viable integrity bug when the codebase or upstream protocol defines exactly how that summary must be chosen. If the authoritative output is the full key list and no singular-target invariant exists, an order-dependent summary is not enough on its own.
