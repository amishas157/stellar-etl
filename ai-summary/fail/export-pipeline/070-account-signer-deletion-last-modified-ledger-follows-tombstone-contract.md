# H070: Synthetic signer deletions should use the deletion ledger as `last_modified_ledger`

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: account-signer history metadata
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `TransformSigners()` synthesizes a deleted `account_signers` row for a signer removed during an account update, the tombstone should use the ledger that performed the deletion as `last_modified_ledger`. A signer deleted in ledger `L` should therefore export `ledger_sequence = L` and `last_modified_ledger = L`, matching the ledger that actually changed signer state.

## Mechanism

The hotfix deletion branch reads signer data from `ledgerChange.Pre` and stamps the synthetic tombstone with `Pre.LastModifiedLedgerSeq` instead of the current updated account's `LastModifiedLedgerSeq`. That initially looks like a stale-metadata bug because the signer is being removed now, not in the earlier ledger that last modified the pre-image.

## Trigger

Export `account_signers` for a ledger where `SetOptions` removes a signer from an existing account and compare the deleted row's `ledger_sequence` against `last_modified_ledger`.

## Target Code

- `internal/transform/account_signer.go:57-85` — synthetic deletion rows use `preLastModifiedLedger`
- `internal/transform/account_signer_test.go:257-289` — test fixture explicitly expects deleted rows to use Pre's `LastModifiedLedgerSeq`
- `internal/transform/account_test.go:190-197` — deleted account rows preserve the removed entry's historical `LastModifiedLedger`
- `internal/transform/offer_test.go:180-185` — deleted offer rows follow the same tombstone convention

## Evidence

The synthetic deletion path sets `LastModifiedLedger: preLastModifiedLedger` even though the row's `LedgerSequence` and `ClosedAt` come from the current deletion ledger. That creates an apparent split between "when this tombstone was emitted" and "what ledger last modified the signer state."

## Anti-Evidence

Checked-in tests explicitly codify this behavior, and other deleted-row exporters in the codebase also preserve the removed entry's historical `LastModifiedLedger` rather than rewriting it to the deletion ledger. The signer tombstone is therefore consistent with the repository's broader tombstone contract.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated in `export-pipeline`

### Why It Failed

The repository already treats `last_modified_ledger` on deleted rows as a property of the removed pre-image, not as "the ledger that emitted the tombstone." `TransformSigners()` matches that existing convention, and the unit test added in the hotfix intentionally locks it in.

### Lesson Learned

When evaluating tombstones, distinguish "removal event metadata" from "last modified state metadata." If sibling deleted-row exporters preserve the pre-image value and tests document the same rule, an apparent stale field is usually contract, not corruption.
