# H016: Pool-share `asset_type` strings are table-specific conventions, not one canonical enum

**Date**: 2026-04-15
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the ETL promises a single canonical `asset_type` vocabulary across all exported tables, then liquidity-pool-share assets should use the same string everywhere so downstream joins and aggregations do not split one asset class into multiple labels.

## Mechanism

This looked plausible because `TransformTrustline()` hard-codes `asset_type = "pool_share"`, while sibling transform paths such as operation details use `asset_type = "liquidity_pool_shares"`. A consumer expecting cross-table normalization could treat those as different asset classes even when they describe the same pool-share instrument.

## Trigger

Compare a pool-share trustline row from `trust_lines` with a pool-share operation-details payload from `history_operations` or effect details from `history_effects`.

## Target Code

- `internal/transform/trustline.go:47-76` — hard-codes `asset_type = "pool_share"` for pool-share trustlines
- `internal/transform/trustline_test.go:149-165` — tests pin `pool_share` for trustline rows
- `internal/transform/operation.go:390-399` — operation helper uses `liquidity_pool_shares`

## Evidence

The repository currently exports at least two different spellings for the same broad asset class, and the trustline tests lock in `pool_share` explicitly. That makes the mismatch easy to observe and tempting to classify as silent structural drift.

## Anti-Evidence

The codebase does not define a single global asset-type enum shared across tables, and existing tests intentionally pin the trustline spelling. Related investigations in effects already concluded that differing pool-share labels across other tables are inherited/table-specific conventions rather than a local corruption bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not establish a codebase contract requiring cross-table `asset_type` normalization. The current repository treats these strings as per-table output conventions, and the trustline spelling is explicitly regression-tested.

### Lesson Learned

String mismatches across tables are only viable when the project defines a shared normalization contract or a shared helper that one path violates. Without that contract, a naming difference — even a confusing one — is not enough to classify as data corruption.
