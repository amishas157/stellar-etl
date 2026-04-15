# H071: Synthetic signer deletions should null out `sponsor` after removal

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: account-signer sponsorship history
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a signer is removed from an account, the deleted `account_signers` tombstone row could be expected to show `sponsor = null`, because after removal there is no surviving signer entry left to be sponsored. A deletion row emitted for signer removal should not imply the signer still exists with an active sponsor after the delete.

## Mechanism

The deletion branch reconstructs tombstones from `ledgerChange.Pre` and copies the removed signer's pre-removal sponsor into the deleted row. That initially looks like a historical mismatch because the row is emitted as `deleted = true`, yet it still carries a non-null sponsor value from the vanished pre-image.

## Trigger

Export `account_signers` for a ledger where a sponsored signer is removed via `SetOptions` and inspect the synthetic deleted row. The tombstone will show `deleted = true` alongside the pre-removal sponsor address.

## Target Code

- `internal/transform/account_signer.go:63-85` — deleted signer rows copy sponsor from `preSponsors`
- `internal/transform/account_signer_test.go:262-289` — current tests preserve pre-removal sponsor semantics for deleted/remaining rows
- `internal/transform/account_test.go:190-197` — deleted account rows preserve sponsor on tombstones
- `internal/transform/offer_test.go:180-185` — deleted offer rows also preserve the removed row's sponsor
- `internal/transform/claimable_balance_test.go:123-129` — deleted claimable-balance rows keep sponsor populated

## Evidence

The signer hotfix reads sponsor data exclusively from the pre-image and does not clear it for deleted rows. In isolation, that looks like a misleading post-delete attribute because the signer entry no longer exists after the update.

## Anti-Evidence

The same pre-image-preserving sponsorship pattern appears across multiple deleted-row exporters, not just account signers. The codebase consistently models tombstones as "last known state plus deleted marker," so a non-null sponsor on a deleted row is established repository behavior rather than a signer-specific bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated in `export-pipeline`

### Why It Failed

Deleted-row exporters throughout the repository preserve sponsorship from the removed pre-image instead of nulling it out. `TransformSigners()` follows that same last-known-state tombstone model, so the non-null sponsor is consistent with existing output contracts.

### Lesson Learned

Before treating a tombstone field as "should be null after deletion," compare how other deleted-row tables encode historical state. In this codebase, tombstones generally preserve the removed row's final known attributes and rely on `deleted = true` / `ledger_entry_change = REMOVED` to signal deletion.
