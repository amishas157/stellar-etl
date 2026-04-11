# H018: Deleted signer rows use the prior last_modified_ledger by design

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If synthetic signer-deletion rows are meant to describe the ledger where the signer was removed, they would be expected to use the account's post-update ledger metadata or the enclosing ledger sequence rather than the pre-update `LastModifiedLedgerSeq`. A deleted signer row would then point at the removal ledger instead of the signer's last live ledger.

## Mechanism

`TransformSigners()` computes deletion rows by diffing `Pre` and `Post` signer sets inside an account update, so it initially looks like `LastModifiedLedger` may be stale when it is copied from `ledgerChange.Pre`. That would make deleted signer rows appear older than the removal event and could confuse history consumers that treat `last_modified_ledger` as the deletion ledger.

## Trigger

Process an updated account entry where signer B exists in `Pre` but not in `Post`, causing the transform to emit a synthetic deletion row for the missing signer.

## Target Code

- `internal/transform/account_signer.go:54-85` — deletion rows are synthesized from `Pre` and use `preLastModifiedLedger`
- `internal/transform/account_signer_test.go:257-263` — test fixture documents the expected deleted-signer behavior

## Evidence

The implementation explicitly sets `LastModifiedLedger: preLastModifiedLedger` for deleted signer rows, even though the enclosing ledger sequence is newer. That makes the row look like it belongs to the previous signer state rather than the deletion event.

## Anti-Evidence

The transform comments explain that signer removals are embedded in an account `UPDATED` change rather than emitted as standalone ledger-entry removals. The test suite also explicitly expects deleted signers to keep `Pre`'s `LastModifiedLedgerSeq`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The code and tests treat `last_modified_ledger` on synthetic deletion rows as "the last ledger where the signer existed," not "the ledger where the deletion row was emitted." That is an intentional semantic choice, not an accidental stale-state bug.

### Lesson Learned

When this package synthesizes rows that do not correspond to a native ledger-entry removal, `last_modified_ledger` may intentionally preserve the pre-state ledger instead of mirroring the enclosing ledger sequence. Test comments are part of the contract here and can rule out an otherwise-plausible stale-field hypothesis.
